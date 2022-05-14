---
title: Building a Nonprofit Compensation Database
description: >
  Processing IRS Form 990 XML records to compile a database of nonprofit
  compensation for executives, directors, and highly compensated individuals
category: technology
tags: nonprofits, IRS, XML, R, "data cleaning"
---

Federal regulations indicate that tax-exempt (nonprofit) organizations may
not wildly overpay their employees and officers
("[excess benefit transactions](https://ecfr.federalregister.gov/current/title-26/chapter-I/subchapter-D/part-53#p-53.4958-4(a))")
without incurring tax penalties.  The excessiveness of a payment is
[measured against](https://ecfr.federalregister.gov/current/title-26/chapter-I/subchapter-D/part-53#p-53.4958-4(b))
the fair market value of goods or the
["reasonableness" of compensation](https://ecfr.federalregister.gov/current/title-26/chapter-I/subchapter-D/part-53#p-53.4958-4(b)(1)(ii)),
namely "the amount that would ordinarily be paid for like services by like
enterprises (whether taxable or tax-exempt) under like circumstances."
Payments by tax-exempt organizations are
[presumed to be reasonable](https://ecfr.federalregister.gov/current/title-26/chapter-I/subchapter-D/part-53#53.4958-6)
if the terms are approved in advance by an "authorized body" (such as the
board or a committee of the board), the organization references data about
compensation for comparable services and organizations, and the process used
to determine the terms is appropriately documented.  The Internal Revenue
Service (IRS) can rebut this presumption of reasonableness with sufficient
evidence that the payments were excessive.

Download the [nonprofit compensation database](https://doi.org/10.7910/DVN/RGATB3).
{:.margin}

All of that is to say that nonprofit organizations will often find
themselves in need of data about what other organizations are paying their
employees, particularly their highest-paid employees, such as the executive
director or chief executive officer.  How are organizations supposed to find
this information?  One particularly useful source is the publicly available
financial reports that tax-exempt organizations have to file with the IRS
each year ([Form 990](https://www.irs.gov/pub/irs-pdf/f990.pdf)), which
includes the compensation of the organization's officers and highest paid
employees.  Let's make a compensation database!

The IRS [releases Form 990 data](https://registry.opendata.aws/irs990/) in an
[XML format](https://www.irs.gov/e-file-providers/current-valid-xml-schemas-and-business-rules-for-exempt-organizations-modernized-e-file)
for organizations that e-filed their returns.  According to
[IRS summary data](https://www.irs.gov/statistics/soi-tax-stats-annual-extract-of-tax-exempt-organization-financial-data),
83% of Form 990 submissions and 62% of Form 990-EZ submissions were
electronic in 2019, so the XML data cover a sizable proportion of records
for tax-exempt organizations.  Coverage is likely weaker for organizations
with smaller budgets, which are more likely to file 990-EZ.  However,
coverage will likely improve in the coming years, as the 2019
[Taxpayer First Act](https://en.wikipedia.org/wiki/Taxpayer_First_Act) requires
[e-filing for tax-exempt organizations](https://www.irs.gov/newsroom/irs-recent-legislation-requires-tax-exempt-organizations-to-e-file-forms)
for tax years beginning after July 1, 2019.

There are three versions of Form 990: the full form, an abbreviated ("EZ")
form, and a form for private foundations.  (There is also a "postcard" form
for small organizations, but those forms aren't included in the XML
repository, currently.) The forms each put our information of interest in
slightly different places.  Moreover, the IRS has changed the XML format,
most notably in 2013.  So, we'll need to sanitize the compensation data to
make it easier to handle.  [Jacob Fenton](https://github.com/jsfenfen) has
helpfully compiled a table of the
[field descriptions](https://raw.githubusercontent.com/jsfenfen/990-xml-metadata/master/descriptions.csv).
I extracted the
[fields of interest](https://gist.github.com/mculbert/6ea6a6bbfa1227967521a3d255fb2c47)
for the compensation database and labeled each with a target variable name.

Rather than working entirely in memory, we'll process each file
individually, writing out the results to disk, and then load in the compiled
data at the end. This will prevent having to constantly reallocate and copy
large data tables.


# Preparation

To get going, we first load the necessary packages
([`xml2`](https://cran.r-project.org/web/packages/xml2/),
[`httr`](https://cran.r-project.org/web/packages/httr/),
and [`htmltools`](https://cran.r-project.org/web/packages/htmltools/))
and create a couple of helper functions to handle missing
values. We'll read in the mapping of XML elements from the various Form 990
versions to consolidated variable names, and set a `startdate`, which could
be used to limit processing only to recently released records (e.g., to
update the database later as the IRS makes more data available).

<figure markdown="1"><figcaption>Preliminaries</figcaption>
~~~r
library(xml2)
library(httr)
library(htmltools)

# Helper function to remove missing elements
condense <- function(x) x[!is.na(x)]

# Helper function to replace NA with empty string
naToEmpty <- function(x) ifelse(sapply(x, is.na), "", x)

# Read in the variable name mapping
vars <- read.csv('https://gist.github.com/mculbert/6ea6a6bbfa1227967521a3d255fb2c47/raw/aee30441c19ebab6bab2380f733a4487cdbc3630/form-990-compensation-fields.csv', as.is=T)

# Earliest file date to process
startdate <- as.POSIXct("2000-01-01T00:00:00.000Z")

# Base URL for S3 bucket listing
baseurl <- "http://irs-form-990.s3.amazonaws.com/?list-type=2&prefix=2"
~~~
</figure>

Bucket listing
{:.margin}

The URL in `baseurl` asks the [Amazon S3](https://aws.amazon.com/s3/) API to
[list the objects](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html)
in the IRS Form 990 bucket. We specify `prefix=2` to exclude the index files
(the identifiers for the XML files start with the year in which the form was
submitted to the IRS). If we knew that we only wanted returns submitted
starting in a given year, we could also include the `start-after` parameter,
such as "`&start-after=2019`" for forms submitted to the IRS in 2019 or later.

Intermediate output files
{:.margin}

We'll create two output files: The "submission records file" will include a
record for each Form 990 submission; the "compensation records file" will
include a row for each person listed on the Form 990---directors, officers,
and highly compensated individuals.

<figure markdown="1"><figcaption>Submission records file</figcaption>
~~~r
# Open the output file
rec <- file('submissions.txt', 'w')

# XML element to consolidated variable mapping
rec.elems <- paste0("/", gsub("/(?!@)", "/d1:",
  vars$Element[vars$Group == ''], perl=T))
rec.map <- vars$Variable[vars$Group == '']
rec.vars <- unique(rec.map)

# Output the header row
cat(c("id", rec.vars), file=rec, sep="\t");
cat("\n", file=rec)
~~~
</figure>

Mapping
{:.margin}

The mapping file includes the [XPath](https://en.wikipedia.org/wiki/XPath)
for each XML element of interest in the `Element` field and the target
variable name in the `Variable` field. Many data elements, such as the
organization's name and annual expenses, have only one value for each
Form 990 submission, but the elements related to each individual are repeated
for each individual. The `Group` field identifies XML elements that are
repeated in each individual's block. For the submission records file, we want
only one row per Form 990 submission, so we subset to those XML elements that
don't belong to a group (`vars$Group == ''`). The `rec.elems` variable holds
the XPaths for the elements of interest; the `rec.map` variable holds
the corresponding consolidated variable names; and `rec.vars` provides the
order of the variables for the output file.

Default namespace
{:.margin}

The [`xml2` package](https://cran.r-project.org/web/packages/xml2/)
names all default namespaces with `d1`, `d2`, etc.  (see `xml_ns()`), so we
need to add the explicit namespace "`d1:`" to the XPath elements in
`rec.elems`. The call to `gsub()` looks for all forward slashes (`/`) that
are not followed by an attribute name (signified by `@`) and replaces them
with "`/d1:`". We also prepend the path with an extra slash to indicate that
the path does not need to start from the root node (a `Return` element).

Compensation record blocks
{:.margin}

The process is similar for the compensation records file, except that we
have a mapping for each of the different types of blocks for each individual
(`comp.groups`), so `comp.elems` and `comp.map` are lists indexed by the
XPath for the parent node for the block.

<figure markdown="1"><figcaption>Compensation records file</figcaption>
~~~r
# Open the output file
comp <- file('compensation.txt', 'w')

# XML element to consolidated variable mapping
comp.groups <- setdiff(unique(vars$Group), '')
comp.elems <- lapply(comp.groups,
     function(g) vars$Element[vars$Group == g])
comp.map <- lapply(comp.groups,
     function(g) vars$Variable[vars$Group == g])
names(comp.elems) <- names(comp.map) <- comp.groups <-
  paste0("/", gsub("/(?!@)", "/d1:",
         comp.groups, perl=T))
comp.vars <- unique(vars$Variable[vars$Group != ''])

# Output the header row
cat(c("id", comp.vars), file=comp, sep="\t")
cat("\n", file=comp)
~~~
</figure>


# Extraction

Listing pagination
{:.margin}

The Amazon S3 API paginates bucket listings by 1,000 objects, so we'll have
to request a page of results, process those files, and then request the next
page of results, etc. The next page of results is indicated by a
"continuation token" (`NextContinuationToken` in the listing results data
structure) appended to the bucket listing API call.

<figure class="wide" markdown="1"><figcaption>Iterate over submissions</figcaption>
~~~r
err <- NULL       # Files that generate HTTP errors
url <- baseurl    # S3 API URL for bucket listing

# Iterate over each page of the bucket listing
while (!is.null(url)) {

  # Get the next page of S3 bucket contents
  res <- read_xml(content(GET(url), 'text', encoding='UTF-8'))

  # Extract the continuation token for the next page
  cont <- xml_text(xml_find_first(res,
    "/d1:ListBucketResult/d1:NextContinuationToken"))

  # Iterate over each object (Form 990 file) in this page
  for (file in xml_find_all(res, "/d1:ListBucketResult/d1:Contents")) {

    # Check to see if this file is new enough
    if (as.POSIXct(xml_text(xml_find_first(file, "./d1:LastModified")))
        >= startdate) {
      # ... Process submission (see code below) ...
    }
  }

  # Clean up the bucket contents data structure
  rm(res)

  # Have we reached the last page?
  if (is.na(cont))
    url <- NULL
  else
    url <- paste0(baseurl, "&continuation-token=", urlEncodePath(cont))

  # Clean up memory
  gc()
}
~~~
</figure>

Error handling
{:.margin}

Since there are over 3.5 million files to process, we wouldn't want the
script to die in the middle due to an HTTP error, so we'll protect the
fetching and parsing of each XML file in a `tryCatch()` block. If there is
an error downloading a file, we add the `key` to `err` for retrying later.

<figure class="wide" markdown="1"><figcaption>Process a submission</figcaption>
~~~r
# Get the object key from the bucket listing
key <- xml_text(xml_find_first(file, "./d1:Key"))
id <- sub("_public.xml", "", key)

# Fetch the XML file
ret <- tryCatch(read_xml(content(
                GET(paste0("http://s3.amazonaws.com/irs-form-990/", key)),
                'text', encoding='UTF-8')),
              error = function(e) { err <- c(err, key); NULL })
if (is.null(ret)) next

# Extract the desired elements
vals <- sapply(rec.elems, function(e) xml_text(xml_find_first(ret, e)))

# Push data through the element-to-variable mapping
vals <- naToEmpty(condense(setNames(vals, rec.map))[rec.vars])

# Output to the submission records file
cat(id, paste(vals, collapse="\t"), sep="\t", file=rec)
cat("\n", file=rec)

# Find individuals listed on the submission, by block type
for (g in comp.groups) {
  # ... Process individual block type (see code below) ...
}

# Clean up the submission data structure
rm(ret)
~~~
</figure>

Submission record extraction
{:.margin}

We call `xml_find_first()` for each element `rec.elems` to extract the
submissions record data, storing the results (either the data element or
`NULL` if the element wasn't in the file) in the `vals` vector. Then, we
rename the `vals` elements with their target variable names (`setNames(vals,
rec.map)`) and remove the empty elements (`condense()`). We then order the
results according to the column order for our output file
(`(...)[rec.vars]`) and replace missing values with empty strings
(`naToEmpty()`) before outputting the submission record.

Individuals by block type
{:.margin}

The data about individuals (directors, officers, and highly compensated
employees) is stuffed into different elements in the various versions of
Form 990 (and even in different sections of Form 990-EZ). To extract the data
about individuals, we'll iterate over each block type (`comp.groups`).

<figure class="wide" markdown="1"><figcaption>Process individuals</figcaption>
~~~r
# Extract all individuals for this block type (g), collapsing list-of-lists
vals <- lapply(xml_find_all(ret, g),
               function(p) sapply(as_list(p), getElement, name=1))

# Push data through the element-to-variable mapping
vals <- lapply(vals, function(p)
  naToEmpty(condense(
    setNames(p[ comp.elems[[g]] ], comp.map[[g]]))[comp.vars] ))

# Collapse each individual into a single tab-delimited string
vals <- sapply(vals, function(p) paste(p, collapse="\t"))

# Output to the compensation records file
if (length(vals) > 0) {
  cat(paste0(paste(id, vals, sep="\t"), collapse="\n"), "\n", file=comp)
}
~~~
</figure>

Extract individuals
{:.margin}

We start by finding all of the nodes that correspond to block type `g`
(`xml_find_all()`). We convert each node to a list of individual data
elements (`as_list(p)`, note that this is `xml2::as_list()`, not
`base::as.list()`). This list of individual data elements is actually a
list of lists: Each child of the individual block could, in principle, have
multiple children. However, in the context of this particular XML schema,
we know that each child of the individual block will have only a single
value. So, we want to collapse the list of lists so that each element of
the resulting individuals list `vals` is a simple named vector of the
individual compensation elements. To collapse the list of lists, we call
`getElement()` on each of the sublists to extract the first (and only)
element, allowing `sapply()` to combine the results into the simple named
vector. At the end of this line, `vals` is a list of compensation data
vectors for each individual.

Element to variable mapping for individuals
{:.margin}

Next, we want to rename the data elements for each individual from the XML
element names to our target variable names and reorder these variables for
output to the compensation records file. For each individual's named vector
(`p`), we reorder the elements according to our canonical element order for
this block (`g`) from the mapping file (`p[ comp.elements[[g]] ]`) and
rename with our target variable names (`setNames(..., comp.map[[g]])`).
Then, we reorder the elements again according to our canonical column
ordering for the compensation records file (`(...)[comp.vars]`) and replace
missing values with empty strings (`naToEmpty()`). At the end of this line,
`vals` is still a list of compensation vectors for each individual, but the
vectors have been reorganized into the standard column order.


# Clean up

To finish up, we close the two intermediate output files and write out the
vector of files that encountered HTTP errors for follow-up later.

<figure markdown="1"><figcaption>Close files</figcaption>
~~~r
close(rec)
close(comp)
save(err, file='err.Rdata', compress='xz')
~~~
</figure>

We read the data back in using the
[`data.table` package](https://cran.r-project.org/web/packages/data.table/),
which is much more efficient than the usual `data.frame` for very large
datasets.

<figure markdown="1"><figcaption>Reread data</figcaption>
~~~r
library(data.table)

rec <- fread('submissions.txt', header=T, sep='\t', quote='', colClasses='character')
comp <- fread('compensation.txt', header=T, sep='\t', quote='', colClasses='character')
~~~
</figure>

Amended returns
{:.margin}

When an organization files an amended Form 990, the IRS releases both the
original and the amended returns. For this database, we only want to keep
the most recent (amended) data. So, we'll sort the dataset by the submission
`Timestamp` and select the most-recent record for each organization (`EIN`)
and tax year.

<figure markdown="1"><figcaption></figcaption>
~~~r
# Parse the submission date (ISO 8601)
rec[, ts := as.POSIXct(sub("Z", "+0000",
              gsub("(.*):(..)$","\\1\\2",Timestamp)),
              format="%Y-%m-%dT%H:%M:%S%z")]

# Order by EIN, TaxYear, and the submission date
setorder(rec, EIN, TaxYear, ts)

# Select only the most recent record per EIN/TaxYear
rec <- rec[, .SD[.N], by=.(EIN, TaxYear)]

# Set keys for merging
setkey(rec, id, TaxYear)
setkey(comp, id)
~~~
</figure>

Parse ISO 8601 date
{:.margin}

The dates in the XML file are in
[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format, which requires a
little [munging](https://stackoverflow.com/questions/15838548/parsing-iso8601-date-and-time-format-in-r)
before we can parse it with `as.POSIXct()`. Then we order by the parsed
timestamp `ts` and select the last (i.e., most recent) record (`.SD[.N]`)
for each organization and tax year (`by=.(EIN, TaxYear)`).

Finally, we can output the final database. To make handling the data a
little easier, particularly for users who only want data from certain years,
we'll output a file for each tax year separately.

<figure markdown="1"><figcaption>Output final database by year</figcaption>
~~~r
# For each tax year
for (y in rec[, unique(TaxYear)]) {

  # Output the submission records for the given year
  fwrite(rec[TaxYear==y, .(id, EIN, TaxYear, TaxPeriodEnd, ReturnType, Name1, Name2, State,
                           X501c, X501cType, X501c3, X527, X4947a1, X4947a1T, XTaxablePF,
                           Expenses, Assets, NumEmp, NumVol, Timestamp)],
         file=paste0("organization-", y, ".txt"),
         sep="\t", quote=F, na='', row.names=F)

  # Output the compensation records for the given year
  fwrite(comp[rec[TaxYear==y], .(id, PersonName, Title, Hours, Compensation,
                                 Director, Institutional, Officer, KeyEmployee, Highest, Former),
              nomatch=0],
         file=paste0("compensation-", y, ".txt"),
         sep="\t", quote=F, na='', row.names=F)

}
~~~
</figure>

Filtering compensation records
{:.margin}

We filtered the submission records to select the most recent submission
explicitly. For the compensation records, we'll use the retained submission
records as our guide, performing an
[inner join](https://rstudio-pubs-static.s3.amazonaws.com/52230_5ae0d25125b544caab32f75f0360e775.html)
using the `X[Y, nomatch=0]` idiom (see `?"[.data.table"`).

# Performance

Amazon EC2
{:.margin}

Since the IRS XML files are stored in an Amazon S3 bucket in the `us-east-1`
(Northern Virginia) region, I figured the data transfer would go a lot
faster if I processed the data on an
[Amazon EC2](https://aws.amazon.com/ec2/) instance in the same region. (No
need to pull the data halfway across the country!) I used an
[`m6g.medium`](https://aws.amazon.com/ec2/instance-types/m6/)
instance, built with the new custom-to-Amazon Arm-based
[Graviton2 processors](https://aws.amazon.com/ec2/graviton/).

Throughput
{:.margin}

As of early June 2021, the IRS had released 3,534,226 Form 990 records,
which included 33,369,586 listed individuals (directors, officers, highly
compensated employees), yielding a 2.5 GB dataset. Data processing took
{%comment%}235566 s{%endcomment%} 65.4 hours (2.7 days), at a rate of
15 submissions per second. Throughout, the CPU was at about 38% utilization,
suggesting that processing was likely network-I/O bound, even given the
speed advantages of data transfers within the same data center. I probably
could have achieved a 30% speed up by breaking the download and parsing code
into two separate threads (so that the download of the next file could
continue during parsing of the last), but the extra coding effort probably
wouldn't have been worth it.

Memory management
{:.margin}

I originally attempted this process using the
[`XML` package](https://cran.r-project.org/web/packages/XML/), rather than the
[`xml2` package](https://cran.r-project.org/web/packages/xml2/).
Both are based on the [`libxml2`](https://en.wikipedia.org/wiki/Libxml2)
library, but it turns out that the `XML` package leaks memory like crazy,
even using `XML::free()` to (ostensibly) release internal parser structures.
With 3.5 million documents to parse, such rampant memory mismanagement was a
nonstarter (my initial run ground to a halt after about 4 hours on a machine
with 4 GB of RAM). Processing went much more smoothly with `xml2`. Memory
usage was about 6 times larger at the end of the 2.7-day run than at the
beginning, suggesting that there was likely still a leak somewhere, but
total usage stayed under 1 GB for the entire process, so the minor leak was
workable.
