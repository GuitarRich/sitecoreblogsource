---
title: 'SXA: Fixing the robots.txt'
tags:
- Sitecore
- SxA
- SEO
category: SxA
---
SXA has many features that help with SEO, sitemaps, metadata etc... and a handler that supplies the *Robots.txt* file for the site. There is a field that populates the `robots.txt` file, located on the `<tenant>/<site root>/Settings` item, in a field called `Robots content`:

{% asset_img robotstxt.png "Set the robots.txt file content" %}

Just add your content you need for the file in there, and the file is dynamically created per request. Not only that, but SXA automatically adds the `/sitemap.xml` url in for you!

### Sweet, why does it need fixing?

Great question, glad you asked! The problem here is that, this one field controls the content of the `robots.txt` file for all environments, both the `Content Management` and the `Content Delivery` system would have the same `robots.txt` file. This causes a problem when you have a `CM` environment that is not hidden behind a VPN. Lets say we have 2 sub domains, `cm.sitecorenutsbolts.com` is our content management system and `www.sitecorenutsbolts.com` is the public website. Yes, Sitecore does recommend that the content mangaement system be hidden behind a VPN, this is not always possible. The CM environment needs to be available over the internet. In these cases, the `robots.txt` should deny crawlers from crawling the site. The same would be true if there were a staging environment that needed to be available over the internet without a VPN, but should not be indexed in search engines.

With the SXA implementation of the robots.txt file, this is not possible just by filling in the fields because the master and web databases would have the same content.

### Fixing the Handler

The way SXA renders out the `robots.txt` file is via a processor in the `httpRequestBegin` pipeline: `Sitecore.XA.Feature.SiteMetadata.Pipelines.HttpRequestBegin.RobotsHandler`. This processor in turn runs a pipeline called `getRobotsContent`, and this is what gets the content from that settings field based on the site. So lets replace that!

First we need somewhere to put the `robots.txt` content. The simplest way to have different content per domain would be to use the Site Grouping items. Normaly you would have a few of these items to resolve your CM, CDs and maybe any staging areas. Also, it allows us to create insert options using the Rules Engine, so no OOTB templates have to be altered.

{% asset_img sitegrouping.png "Site Grouping Setup" %}

So lets create a template for the header file, nice and simple, just a single field required:

