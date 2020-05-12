---
title: Working with External Data - SXA and Wildcards
tags:
- Sitecore
- SxA
- External Data
category: SxA
---

If you have implemented Sitecore for a while, you will have come across a requirement to integrate external data within the Sitecore delivered pages. There are a few options that you can use to integrate this data, from importing the data into Sitecore Items (pro tip, don't do this), to storing the data in indexes or other external databases. But with each option, as the implementer you will have to decide how Sitecore is going to resolve a url and read the external content.

This blog will not attempt to address the "where" to store this data, but it will show some options to resolve the url and make the content available for components. It will be SXA focused, but the code is not SXA specific and could be used in any implementation.

### Building the URL/Wildcard Items

Most Sitecore developers have heard of `Wildcard Items`, but here is a quick refresher for those that haven't. If you create a new item and name it `*`, that becomes a catch all `wildcard` item. Sitecore will resolve any text in that segment to that item. 

Its important to know that wildcard items only relate to a single segment in the Url. For example, if we have the following tree:

<!-- Insert tree picture here -->

The item path is `/sitecore/content/my-tenant/my-site/home/films/*`. This would resolve any word after the segment `films` in the url. E.g:

- https://www.my-film-site.com/films/star-wars-a-new-hope
- https://www.my-film-site.com/films/star-wars-the-empire-strikes-back
- https://www.my-film-site.com/films/star-wars-return-of-the-jedi

_BUT_ It is not a catch all for _everything_ after the `films` segment. So these urls would still give you a 404:

- https://www.my-film-site.com/films/star-wars/a-new-hope
- https://www.my-film-site.com/films/star-wars/the-empire-strikes-back
- https://www.my-film-site.com/films/star-wars/return-of-the-jedi

To make the second list work, we would need multiple wildcards in the tree:

`/sitecore/content/my-tenant/my-site/home/films/*`
