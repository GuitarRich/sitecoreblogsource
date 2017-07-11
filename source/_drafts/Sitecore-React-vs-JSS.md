---
title: Sitecore.React vs JSS
tags:
- Sitecore
- Sitecore JSS
- Sitecore.React
category: Sitecore.React
---
{% asset_img elephant-in-the-room.gif %}

Ok - so lets address the new elephant in the room! [Sitecore JSS](https://twitter.com/hashtag/SitecoreJSS?src=hash) - Sitecore JSS is a hugely exciting project for Sitecore devs that are looking toward using front end based frameworks like ReactJS, Angular, Backbone etc... It promises to revolutionise Sitecore developement... for some!

So with Sitecore JSS in the pipeline, where does that leave the [Sitecore.React]() module? Well I think it still has its place and rather than being a competetor to Sitecore JSS, it sits next to it in the market. I'll try to explain why:

# Sitecore JSS or Sitecore.React - Which one should you use?

## .Net vs Node

One of the main requirements of the Sitecore.React module was that in production I did not want to have to worry about anything extra to a standard Sitecore installation. So this meant a standard .Net stack - Windows Server, SQL Server, IIS, .Net framework etc... - sure Sitecore introduces MongoDb and SOLR, but for the delivery servers, its still a pretty standard stack.

So with Sitecore.React I did not want to introduce any node.js servers.

Sitecore JSS requires node.js servers running side alongside the delivery servers to deliver the site. 
