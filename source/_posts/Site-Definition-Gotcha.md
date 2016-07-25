title: "Site Definition Gotcha"
date: 2016-03-18 02:04:01
tags:
category: Sitecore
---

Don't you just love those problems that end up with you tearing your hair out! I ran into just such a problem this week with a client. Let me set the scene...

{% asset_img hours.jpg %}

The client in question has a fairly simple multisite setup. Some of the content is shared between sites, other content is local to each site. Also, because the sites are all in the same family, Site B can list articles from Site A. But these article listings must provide links back to the article page on Site A, not display them with Site B's domain. Should be simple right...

Now Sitecore does have settings for site resolving and the link provider should be able to handle this. But I have found that just setting the `SiteResolving` setting does not always work. Others have also found this ([Sitecore Link Manager Url Options Site Resolving is not Set](http://stackoverflow.com/questions/29774360/sitecore-link-manager-url-options-site-resolving-is-not-set)). Also the site already has a custom link provider, so we modified the link provider to check if the item was part of the current site. If it was not, make sure the `SiteResolving` flag was set correctly and then generate the Url. We also set the `targetHostName` to make sure that the urls are generated with the correct domain.

All was well... or so we thought. We did our local testing and sure enough navigating to **Site B** and loading the page with the article listing from **Site A** rendered links pointing to `http://sitea.local/news/article-name` - perfect. Run through a few more tests and its all looking great. Code gets committed and deployed out to the internal QA site.

####All was not well!!
After deploying and making sure everything was published, all the indexes were updated etc... I hit the page on **Site B** that listed articles from **Site A**... problem... the links were pointing to the content, but with **Site B**'s domain name.

All the config looked ok. The custom link provider was being used, the Site Definitions had the `targetHostName` and `host` values set correctly. Everything matched my local environment, but it was not working. I double checked my local and that did work.

Eventually after dotPeek'ing through the link provider in Sitecore I narrowed it down to this method in `Sitecore.Links.LinkProvider.LinkBuilder`

```
protected virtual string GetServerUrlElement(SiteInfo siteInfo)
{
    SiteContext site = Context.Site;
    string str1 = site != null ? site.Name : string.Empty;
    string hostName = this.GetHostName();
    string str2 = this.AlwaysIncludeServerUrl ? WebUtil.GetServerUrl() : string.Empty;
    if (siteInfo == null)
        return str2;
    string str3;
    if (string.IsNullOrEmpty(siteInfo.HostName) || string.IsNullOrEmpty(hostName) || !siteInfo.Matches(hostName))
        str3 = StringUtil.GetString(new string[2]
        {
        this.GetTargetHostName(siteInfo),
        hostName
        });
    else
        str3 = hostName;
    string str4 = str3;
    if (!this.AlwaysIncludeServerUrl && siteInfo.Name.Equals(str1, StringComparison.OrdinalIgnoreCase) && hostName.Equals(str4, StringComparison.OrdinalIgnoreCase) || (str4 == string.Empty || str4.IndexOf('*') >= 0))
        return str2;
    string @string = StringUtil.GetString(new string[2]
    {
        siteInfo.Scheme,
        this.GetScheme()
    });
    int @int = MainUtil.GetInt(siteInfo.Port, WebUtil.GetPort());
    int port = WebUtil.GetPort();
    string scheme = this.GetScheme();
    StringComparison comparisonType = StringComparison.OrdinalIgnoreCase;
    if (str4.Equals(hostName, comparisonType) && @int == port && @string.Equals(scheme, comparisonType))
        return str2;
    string str5 = @string + "://" + str4;
    if (@int > 0 && @int != 80)
        str5 = str5 + (object) ":" + (string) (object) @int;
    return str5;
}
```

And more specifically, this line:

```
if (string.IsNullOrEmpty(siteInfo.HostName) || string.IsNullOrEmpty(hostName) || !siteInfo.Matches(hostName))
```

On my local, that line always returned true - and the `targetHostName` was used for the server url part. But on the QA servers, this always returned false.

The part that was giving me the trouble was `!siteInfo.Matches(hostName)` - This check tries to match the `HttpContext.Current.Request.Url.Host` with the values set in the Site Definition `host` property. If there is a match, then it assumes the `hostName` *is* the right domain for the item and generates the link under that *hostName*. If there is *NO* match, it uses the `targetHostName` property to set the domain for the Url.

On my local, the site definitions were:

```xml
<site name="siteB" patch:before="site[@name='website']"
        hostName="siteb.local"
        virtualFolder="/"
        physicalFolder="/"
        rootPath="/sitecore/content/siteB"
        startItem="/home"
        database="web"
        content="master"
        language="en"
        targetHostName="siteb.local"
        scheme="https" />

<site name="siteA" patch:before="site[@name='website']"
        hostName="sitea.local"
        virtualFolder="/"
        physicalFolder="/"
        rootPath="/sitecore/content/siteA"
        startItem="/home"
        database="web"
        content="master"
        language="en"
        targetHostName="sitea.local"
        scheme="https" />
```

but on the QA servers, we has a slightly different setup - see if you can see what caused us the problem:

```xml
<site name="siteB" patch:before="site[@name='website']"
        hostName="qa-siteb.sitea.com|siteb.sitea.com"
        virtualFolder="/"
        physicalFolder="/"
        rootPath="/sitecore/content/siteB"
        startItem="/home"
        database="web"
        content="master"
        language="en"
        targetHostName="siteb.local"
        scheme="https" />

<site name="siteA" patch:before="site[@name='website']"
        hostName="*.sitea.com"
        virtualFolder="/"
        physicalFolder="/"
        rootPath="/sitecore/content/siteA"
        startItem="/home"
        database="web"
        content="master"
        language="en"
        targetHostName="sitea.local"
        scheme="https" />
```

The problem here was the wildcard Url for SiteA. On the QA environment, we only had the domain for `sitea.com`. **Site B** was setup as a sub-domain of Site A. This meant that when the LinkProvider checked to see if the `hostName`, which was `http://siteb.sitea.com/` matched the `host` property on `Site A` definition, it returned true, because the wildcard host `*.sitea.com` was a valid match.

####Moral of the story or TL/DR;
If you have a multisite setup and some sites use subdomains of the main site. Becareful with using wildcard matches in the `host` property. For us the fix was a simple change, `*.sitea.com` became `qa.sitea.com` and the cross site links started working on the QA site just as they had done locally.

Hopefully this helps someone else keep thier hair and not spend an afternoon feeling like you are going crazy! :D

Of course you could always have this attitude:

{% asset_img workedin.jpg %}

-- Richard

