---
title: Custom Fonts Not Downloading in SXA Theme
tags:
  - Sitecore
  - SxA
  - Themes
category: SxA
date: 2019-08-09 15:34:39
---

A collegue of mine ran into a weird problem recently with getting a custom font to download properly in his SXA site theme. Once I looked at the theme items, I realized the problem right away, its a problem that I ran into and I have helped out a number of developers on [Sitecore Slack](https://sitecore.chat) with the same problem, so I thought it time to have a more permanent record of this:

## The problem with fonts in SXA

Here is a pretty typical setup for a `font-face` definition in your sass files:

```sass
@font-face {
    font-family: 'Gotham';
    src: url('../fonts/gotham/gotham-medium.eot');
    src: url('../fonts/gotham/gotham-medium.eot?#iefix') format('embedded-opentype'),
    url("../fonts/gotham/gotham-medium.woff") format("woff"),
    url("../fonts/gotham/gotham-medium.otf") format("opentype")
}

@font-face {
    font-family: 'Gotham-Bold';
    src: url('../fonts/gotham/gotham-bold.eot');
    src: url('../fonts/gotham/gotham-bold.eot?#iefix') format('embedded-opentype'),
    url("../fonts/gotham/gotham-bold.woff") format("woff"),
    url("../fonts/gotham/gotham-bold.otf") format("opentype")
}

@font-face {
    font-family: 'Gotham-Black';
    src: url('../fonts/gotham/gotham-black.eot');
    src: url('../fonts/gotham/gotham-black.eot?#iefix') format('embedded-opentype'),
    url("../fonts/gotham/gotham-black.woff") format("woff"),
    url("../fonts/gotham/gotham-black.otf") format("opentype")
}

@font-face {
    font-family: 'Gotham-Book';
    src: url('../fonts/gotham/gotham-book.eot');
    src: url('../fonts/gotham/gotham-book.eot?#iefix') format('embedded-opentype'),
    url("../fonts/gotham/gotham-book.woff") format("woff"),
    url("../fonts/gotham/gotham-book.otf") format("opentype")
}

```

And here are the corresponding font files:

{% asset_img gothamfiles.png "Gotham Font Files" %}

So far, nothing strange about that, and in a classic Sitecore site where the files are stored on the file system, that would be all you need to do. In fact, if your FED team are styling the theme using the static html files exported by Creative Exchange, everything will look great.

But now lets import that theme into Sitecore and all of a sudden the font files stop downloading properly. The mime type is wrong and the browser can't decode the fonts. So what gives?

Let's look at the items created in Sitecore and see if you can spot the problem:

{% asset_img gothamsitecoreitems.png "Gotham Font Files as Sitecore Items" %}

Spot the problem? When the files are uploaded to Sitecore, the file extension is stripped off and just added as meta data on the media item. When the browser makes a request to `../fonts/gotham/gotham-medium.woff, Sitecore ignores the `.woff` extension and tries to resolve `gotham-medium` to an item. As there are 7 items called `gotham-medium` in the folder, Sitecore will always return the first one it finds. Which in my case was `gotham-medium.eot` and that is not going to work in Chrome!

## Rename Your Font Files

The simple answer here is to append the extension to the font file before uploading:

{% asset_img gothamsitecoreitemsfixed.png "Fixed font file names" %}

and update your sass to reference the new font names, now Sitecore know's which font file you want.

Hope this helps someone who might be having a similar problem!

-- Richard
