---
title: 'Sitecore SXA: Using HTTP Headers to Secure Your Site'
tags:
  - Sitecore
  - SxA
  - Security
category: SxA
date: 2018-07-27 14:35:53
---

I recently came across a nice post on [Using HTTP Headers to Secure Your Site](https://blog.heroku.com/using-http-headers-to-secure-your-site) by [Caleb Thompson](https://blog.heroku.com/authors/caleb-thompson). Now while the blog post was targeted at Ruby, the principles here apply to any website. Using [Observatory](https://observatory.mozilla.org/) by mozilla, we can scan our site and using the results see what we need to configure to help with the security of the site.

As the project I am working on right now is very security focused, the content here is very relevant. So I looked into how we can configure this in a Sitecore SXA build. From that came a new module which I will host on my github account: [SXA Security Headers](https://github.com/GuitarRich/SXA.SecurityHeaders). As soon as Sitecore approves the module on the market place I will post a link here too.

Now, this is not a completely new subject for Sitecore builds, both [Akshay Sura](https://twitter.com/akshaysura13) and [Bas Lijten](https://twitter.com/BasLijten), addressed this a couple of years ago, their posts and module also served as inspiration for this:

* [Secure Sitecore : Secure Headers XSS Protection](https://www.akshaysura.com/2016/08/19/secure-sitecore-secure-headers-xss-protection/)
* [Secure Sitecore : Headers are a headache but nothing we cannot solve!](https://www.akshaysura.com/2016/08/02/secure-sitecore-headers-are-a-headache-but-nothing-we-cannot-solve/)
* [Sitecore Security #3: Prevent XSS using Content Security Policy](http://blog.baslijten.com/sitecore-security-3-prevent-xss-using-content-security-policy/)

But my requirements needed a way have having this in a multisite environment, with custom headers per site. SXA does multisite very well, so lets follow their pattern.

## What do the Security Headers actually do?

So lets first look at what security headers we will be adding and a quick overview of what they are for. I'm not going into detail here, because there are many resources available already for this:

#### Content Security Policy (CSP)

This is a pretty well known header. CSP gives us control over where scripts and resources we reference on our site can be loaded from. This normally would include your own site, and maybe a few external CDN sites, like Google Tag Manager. The header can be broken down to protect various types of assets, such as `scripts`, `images`, `css` etc...

A list of the policy options for the Content Security Policy and what each one does can be found here: [Mozilla: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).

This header also helps to protect against XSS attacks, by controlling who is allowed to load scripts on to our site. You can add the source for `frame-ancestors`, this prevents our site from being loaded into an iFrame on any site, except those that we allow. It helps prevent [clickjacking attacks](https://www.owasp.org/index.php/Clickjacking).

#### HTTP Strict Transport Security (HSTS)

HSTS tells the browser that our site should _only_ be viewed over https. The details of the attributes are on the [OWASP Cheat Sheet for HSTS](https://www.owasp.org/index.php/HTTP_Strict_Transport_Security_Cheat_Sheet). Enable this if you only want https traffic... which... should be everyone by now!

#### X-Content-Type-Options

This will tell the browser not to load scripts or styleshees if the MIME type as indicated by the server is not correct.

#### X-Frame-Options

This is a fallback for older browsers that do not obey the `frame-ancestors 'none'` CSP. It will prevent your site from being loaded into an iFrame.

#### X-XSS-Protection

Again, a fallback for older browser and provides similar protection against XSS attacks as the CSP will.

#### Referrer Policy

The referrer policy defines whether the referrer header is sent to the destination when a user clicks a link.

## Lets do this with SXA Security Headers

So how does the module work? The module works on a Site basis, so if you are multisite, you can specify your security headers per site. It adds a new `Settings` item that contains child items for each of the above headers.

{% asset_img templates.png "SXA Security Headers Templates" %}

Each of these items is responsible for setting a security header when added to an SXA Site.

### Installing the module

To install the module, just download the package and install using the Sitecore Package Installer. It is setup to follow the standard SXA patterns, so once installed, you can right click your Tenant/Site and add the module. 

Once the package is installed you will be able to add the module to your tennent and then site. Once it is added, navigate to your SXA Site Settings item. A new settings item called  `Security Headers` will have been created. An insert option rule included in the package will enable the right-click insert ability:

{% asset_img install.png "Settings Item after install" %}

Once you have that, you can select which security headers you want to include in the site. Simply, right-click the **Security Headers** item, go to insert, and select from the available options.

A basic set of default header items will be created when you add the feature to your site.

### Content Security Policy

The Content Security Policy header allows you to either enforce or just report violations. Set the checkbox on the item. To configure the different CSP options, you must insert a child item for each source type you want to control. Right-Click the CSP item and insert a **New Policy**:

{% asset_img newpolicy.png "Insert new Content Security Policy" %}

This runs a Sitecore PowerShell script that displays a dialog, giving you all the available options for the policies. It also detects which policies you have already added and removes those from the list. Simply select the required policy and then set the hosts you want to allow on your site. 

{% asset_img newpolicywizard.png "Select a policy type" %}

The module comes with a set of predefined hosts/options that you can select from, but there is also a free text field for you to add any extras:

{% asset_img policyoptions.png "Content Security Policy Options" %}

### Other Headers

The other insert options will give you the ability to add the following headers to your site:

* HTTP Strict Transport Security (HSTS)
* X-Content-Type-Options
* X-Frame-Options
* X-XSS-Protection
* Referrer Policy

For each item there are a set of fields ranging from a simple, Enable/Disable checkbox, to a field to set the required value of the header.

## Useful?

Hopefully someone else will find this useful, although this is a problem that has options already in the Sitecore world, with the new world of SXA and multi-tennancy/multi-sites, this should provide enough flexibility to help make our implementations secure. This is only one of the many steps we may need to take, but its something that we should all be implementing for our clients.

Let me know your thoughts.


