---
title: Lightweight Cookie Consent for Jekyll
description: >
  A simple inline solution for obtaining user consent for Google Analytics
  and Adsense cookies.
category: technology
tags: jekyll Google cookies javascript
---

When it came time to add a cookie consent banner, I wanted something
lightweight—what a shame it would be to gum up a nice, clean
[Jekyll](https://jekyllrb.com/) site with a bunch of bloated Javascript! The
[Jekyll Codex cookie consent](https://jekyllcodex.org/without-plugin/cookie-consent/)
was close, but I wanted to give visitors the option of customizing which
cookies they wanted to accept (e.g., website analytics vs. ad
personalization). The Jekyll Codex example also functions by *reloading the
page* with the cookie code when the visitor consents, which struck me as
unfortunately cumbersome—I would rather have a solution that invisibly works
with the underlying JavaScript in the background so as not to disrupt the
user experience.

[Documentation](https://developers.google.com/gtagjs/devguide/consent)
for `gtag` consent mode
{:.margin}

Happily, Google recently released a beta for a new
["Consent Mode" feature](https://blog.google/products/marketingplatform/360/measure-conversions-while-respecting-user-consent-choices/)
that makes this process easy. With the consent mode feature, you specify a
default for whether cookies are used for ads or for analytics, which you can
then update when a visitors consent or changes their preferences with a simple
call to `gtag()`. You can even specify
[different defaults by region](https://developers.google.com/gtagjs/devguide/consent#region-specific_behavior),
for example to comply with strict consent-first regulations in Europe while
applying a more liberal opt-out policy in other jurisdictions.

Here's what I came up with. First, we have the code for the banner itself:

<figure class="wide" markdown="1"><figcaption>Consent banner block</figcaption>
~~~html
<div id="cookie-consent" aria-label="Cookie consent banner">
  <span style="padding:.5rem;display:block;">We use cookies to understand how
    visitors are using this site and to improve the experience of ads.</span>
  <button id="cookie-consent-dismiss" class="cookie-consent-btn"
          style="background:green;color:white"
          onclick="recordConsent();
                   document.getElementById('cookie-consent').style.display = 'none';">
    OK</button>
  <a class="cookie-consent-btn" href="/privacy">Learn More</a>
  <button id="cookie-consent-customize-btn" class="cookie-consent-btn"
          onclick="document.getElementById('cookie-consent-customize').style.display = 'grid';
                   document.getElementById('cookie-consent-customize-btn').style.display = 'none';">
    Customize</button>
  <div id="cookie-consent-customize"
       style="display:none; padding-top:.5rem; grid-gap:5px;
              grid-template-columns: max-content max-content;">
    <span style="margin-bottom: 10px">Select which cookies you accept:</span>
    <span></span>
    <span id="cookie-consent-label-analytics" aria-hidden="true">Website usage statistics</span>
    <label class="switch"><input id="cookie-consent-analytics" type="checkbox" checked aria-labelledby="cookie-consent-label-analytics" /><span class="slider"></span></label>
    <span id="cookie-consent-label-ads" aria-hidden="true">Ad personalization</span>
    <label class="switch"><input id="cookie-consent-ads" type="checkbox" checked aria-labelledby="cookie-consent-label-ads" /><span class="slider"></span></label>
  </div>
</div>
~~~
</figure>

Even though I style the banner to appear at the bottom of the screen, the
code needs to go at the top of the `<body>` so that screen readers will
provide the consent content to visitors before launching into the rest of
the page. Similarly, the `aria-label` for the consent block will be
important when we get to the button that allows visitors to change their
settings.

The banner consists of a brief explanatory message and three buttons: a
button to accept the cookies, a link to the privacy policy, and a button to
customize which cookies the user wants to accept or deny. These are followed
by a customization block (`cookie-consent-customize`) with controls for each
type of cookies (here, website analytics and ad personalization). The
customization block is hidden by default.

Clicking on the consent button calls the `recordConsent()` function (below)
to record the visitor's choices and hides the consent block. Clicking on the
"Customize" button displays the customization block and hides the "Customize"
button, since it no longer has a function once the customization block is
displayed.

The selectors for each cookie type are courtesy of
[w3schools](https://www.w3schools.com/howto/howto_css_switch.asp).
The slider simply consists of a filled background block of class `.slider`
(to create the track for the switch) and an attached filled foreground block
(to represent the toggle) implemented with `.slider:before`. The slider is
linked with an invisible checkbox input. When the input is checked, the toggle
shape is translated to the right and the background color shifts from grey to
green. An added shadow indicates when the control has the keyboard focus
(made possible with `tab-index: 0`). This is all accomplished via CSS:

<figure markdown="1"><figcaption>Toggle switch stylesheet</figcaption>
~~~ css
.switch {
  position: relative;
  display: inline-block;
  width: 48px; height: 28px;
  tab-index: 0;
}
.switch input { opacity: 0; width: 0; height: 0; }
.slider {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  cursor: pointer;
  background-color: #777777;
  border-radius: 34px;
  transition: .4s;
}
.slider:before {
  position: absolute;
  content: "";
  height: 20px; width: 20px;
  left: 4px; bottom: 4px;
  background-color: white;
  border-radius: 50%;
  transition: .4s;
}
input:focus + .slider { box-shadow: 0 0 20px white; }
input:checked + .slider { background-color: green; }
input:checked + .slider:before { transform: translateX(20px); }
~~~
</figure>

To get the banner to stick to the bottom of the screen, we use
`position: fixed`, and `z-index: 999` places the banner on top of everything
that scrolls underneath it:

<figure markdown="1"><figcaption>Consent banner stylesheet</figcaption>
~~~ css
#cookie-consent {
  background: black;
  color: white;
  width: 100%;
  padding: 1rem;
  position: fixed;
  bottom: 0; left: 0;
  z-index: 999;
}
@media print { #cookie-consent { display: none; } }
~~~
</figure>

The `recordConsent()` function sets three cookies: one for each consent type
(analytics and ads), as well as a convenience cookie that indicates that the
consent banner has been dismissed. It then calls `gtag()` with the updated
consent settings:

<figure markdown="1"><figcaption>recordConsent() function</figcaption>
~~~ javascript
function recordConsent() {
  var date = new Date();
  // One year from now
  date.setTime(date.getTime() + (365*24*3600*1000));
  var suffix = "; expires=" + date.toUTCString() + "; path=/";

  document.cookie = "cookie_consent_ads=" +
    document.getElementById('cookie-consent-ads').checked +
    suffix;
  document.cookie = "cookie_consent_analytics=" +
    document.getElementById('cookie-consent-analytics').checked +
    suffix;
  document.cookie = "cookie_consent_cleared=true" + suffix;

  gtag('consent', 'update', {
    'ad_storage':
      (document.getElementById('cookie-consent-ads').checked ?
       'granted' : 'denied'),
    'analytics_storage':
      (document.getElementById('cookie-consent-analytics').checked ?
       'granted' : 'denied')
  });
}
~~~
</figure>

Now for the part that links it all together. We augment the typical `gtag`
code with default consent settings, then check to see if the user has
dismissed the consent banner. If so, we apply the user's consent settings.
All of this happens before calling `gtag('config')` so that any saved
settings from a previous page view will apply right out of the gate:

<figure class="wide" markdown="1"><figcaption>Apply the saved consent settings</figcaption>
~~~ javascript
window.dataLayer = window.dataLayer || [];
function gtag(){dataLayer.push(arguments);}
gtag('consent', 'default', { 'ad_storage': 'denied' });
gtag('consent', 'default',
      { 'analytics_storage': 'denied', 'region': ['EU', 'UK'] });

if (readCookie('cookie_consent_cleared')) {
  document.getElementById('cookie-consent').style.display = 'none';
  document.getElementById('cookie-consent-ads').checked =
    (readCookie('cookie_consent_ads') == 'true');
  document.getElementById('cookie-consent-analytics').checked =
    (readCookie('cookie_consent_analytics') == 'true');

  gtag('consent', 'update', {
    'ad_storage':
      (document.getElementById('cookie-consent-ads').checked ?
        'granted' : 'denied'),
    'analytics_storage':
      (document.getElementById('cookie-consent-analytics').checked ?
        'granted' : 'denied')
  });
}

gtag('js', new Date());
gtag('config', '[INSERT GOOGLE ANALYTICS PROPERTY ID HERE]');
~~~
</figure>

Although Google recommends putting `gtag('config')` in `<head>`, this block
needs to go after the `cookie-consent` block so that the middle section can
read the status of the checkbox elements.

Note: I am not a GDPR legal expert.
{:.margin}

Google Analytics assigns a random identifier to each visitor that is unique
to the specific website being analyzed.  Unless a website designer
explicitly (or negligently) incorporates identifying information into the
analytics stream (say, after the user logs in), these "identifiers" are, for
all intents and purposes, anonymous. I have no idea the identity of user
"1066186681.1615103843" from Parsons, Kansas (population 9,736).
Nevertheless, European courts apparently consider these random identifiers as
"personally identifiable information" for which positive consent is needed
before they can be sent to a third-party for processing under the
[General Data Protection Regulation (GDPR)](https://gdpr.eu/).
As such, for the default settings, I chose to deny analytics cookies only
for visitors from the EU and UK (allowing users from other regions to
opt-out). For ad personalization, in contrast, I chose to wait until users
from any region provide positive consent because some users can find
personalized ads a little creepy.

The `readCookie()` function comes from the
[Jekyll Codex implementation](https://raw.githubusercontent.com/jhvanderschee/jekyllcodex/gh-pages/_includes/cookie-consent.html)
and simply searches through this page's cookies for the named value:

<figure markdown="1"><figcaption>readCookie() function</figcaption>
~~~ javascript
function readCookie(name) {
  var nameEq = name + "=";
  var cookies = document.cookie.split(';');
  for(var i=0; i < cookies.length; i++) {
    var c = cookies[i];
    while (c.charAt(0)==' ')
      c = c.substring(1, c.length);
    if (c.indexOf(nameEq) == 0)
      return c.substring(nameEq.length,c.length);
  }
  return null;
}
~~~
</figure>

Finally, users need a means for revoking their consent once given. This
simply requires redisplaying the customization block of the consent banner
to allow users to change their preferences and re-record the result. I
accomplish this using a simple button on the [privacy](/privacy) page:

<figure class="wide"  markdown="1"><figcaption>Update consent button</figcaption>
~~~ html
<button class="cookie-consent-btn"
        onclick="document.getElementById('cookie-consent-customize').style.display = 'grid';
                 document.getElementById('cookie-consent-customize-btn').style.display = 'none';
                 document.getElementById('cookie-consent').style.display = 'block';
                 document.getElementById('cookie-consent-dismiss').focus();">
  Update cookie settings</button>
~~~
</figure>

When a user clicks the button, the customize block and consent blocks are
displayed, the customize button is hidden, and the keyboard focus is set to
the "OK" button in the consent banner. Setting the focus is important, here,
for visitors who use screen readers. Without this, there would be no
detectable feedback indicating that an action had occurred when the button
was activated, and users might have difficulty knowing that they need to
navigate backward to find the consent banner at the beginning of the page
(DOM-wise). The `aria-label` for the consent banner helps here, as well, to
let the user know that this "OK" button is in the context of the consent
banner.

And there we have it! The package comes in at about 4kB, which could
probably be reduced a little by minification. All in all, a fairly
lightweight solution, I think.
