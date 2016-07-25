title: "Always Include Server Url without a HttpContext"
date: 2015-11-03 11:59:14
tags:
category: Sitecore
---

Recently we had a requirement to update an external API with content information when new items were published. The external source needed general content like titles, images etc... But also needed a full link to the page. The implementation is a multi-site install and any of the sites could send data to the external API.

The plan was to use the `item:saved` event and if the database was the delivery database, add the item to a queue. Then on `publish:end` we can process the queue and send the data.

###AlwaysIncludeServerUrl
Getting the full url to a page is nice and simple in Sitecore, we can just set the `LinkOptions.AlwaysIncludeServerUrl = true;`. 

```csharp
var options = LinkManager.GetDefaultUrlOptions();
options.AlwaysIncludeServerUrl = true;
var url = LinkManager.GetItemUrl(item, options):
```

But by default that uses the current `SiteContext` to get the host name. Because we are running this in an event, the `SiteContext` wouldn't be our delivery site, it would be the `Shell` site. So we need to run the link manager in a different `SiteContext`. We can do this with a `SiteContextSwitcher`. So we created a method to work out the site based on the location of the item and then ran the link manager in that sites context:

```csharp
var site = this.ContentManager.GetSiteFromItem(item);
using (new SiteContextSwitcher(site))
{
	var options = LinkManager.GetDefaultUrlOptions();
	options.AlwaysIncludeServerUrl = true;
	var url = LinkManager.GetItemUrl(item, options):
}
```

To make this work properly, we also need to set the `targetHostName` in our site definitions config:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
	  <sites>
		  <site name="Site1" patch:before="site[@name='website']"
			  hostName="site1.local|www.site1.com"
			  rootPath="/sitecore/content/sites/site1/"
			  startItem="/home"
			  database="web"
			  language="en"
			  allowDebug="true"
			  cacheHtml="true"
			  htmlCacheSize="10MB"
			  enablePreview="true"
			  enableWebEdit="true"
			  enableDebugger="true"
			  disableClientData="false"
			  enableCustomErrors="true"
			  targetHostName="www.site1.com"
			/>
		  <site name="Site2" patch:before="site[@name='website']"
			  hostName="site2.local|www.site2.com"
			  rootPath="/sitecore/content/sites/site2/"
			  startItem="/home"
			  database="web"
			  language="en"
			  allowDebug="true"
			  cacheHtml="true"
			  htmlCacheSize="10MB"
			  enablePreview="true"
			  enableWebEdit="true"
			  enableDebugger="true"
			  disableClientData="false"
			  enableCustomErrors="true"
			  targetHostName="www.site2.com"
			/>
			<!-- etc... -->
		</sites>
  </sitecore>
</configuration>
```

And now everything works nicely - we can generate the correct Urls for our content and pass it over to our 3rd party... Or so I thought. Actually what happened was that the urls being generated did not include the scheme for the site. So instead of getting this:

`http://www.site1.com/page1/` we got `://www.site1.com/page1/`

My first thought was to include the scheme in the `targetHostName`. But that returned `://http://www.site1.com/page1/` - so somewhere in the Sitecore LinkProvider that `://` was being added and Sitecore could not work out the scheme.

###WebUtil.GetScheme - HttpContext tightly coupled again...
So into dotPeek we go, lets have a look at what Sitecore is doing. The `GetItemUrl` method calls `LinkProvider.GetServerUrlElement` to build that server part. This is the code in question:

```csharp
string @string = StringUtil.GetString(new string[2]
{
	siteInfo.Scheme,
	this.GetScheme()
});
string str5 = @string + "://" + str4;
return str5;
```

And `GetScheme` calls `WebUtil.GetScheme()`

```csharp
public static string GetScheme()
{
	HttpContext current = HttpContext.Current;
	if (current != null)
		return current.Request.Url.Scheme;
	return string.Empty;
}
```
Ah - now we see the problem, in the `publish:end` event `HttpContext.Current == null`. So the scheme is always going to be empty! Fortunately, it looks like we can set the scheme using the site definition, this line `siteInfo.Scheme` looks like it should help here. This is in the constructor on the `SiteInfo` class: 

```csharp
this.scheme = StringUtil.GetString(new string[1]
{
	properties["scheme"]
});
```

So now the site definition looks like this:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
	  <sites>
		  <site name="Site1" patch:before="site[@name='website']"
			  hostName="site1.local|www.site1.com"
			  rootPath="/sitecore/content/sites/site1/"
			  startItem="/home"
			  database="web"
			  language="en"
			  allowDebug="true"
			  cacheHtml="true"
			  htmlCacheSize="10MB"
			  enablePreview="true"
			  enableWebEdit="true"
			  enableDebugger="true"
			  disableClientData="false"
			  enableCustomErrors="true"
			  targetHostName="www.site1.com"
			  scheme="http" <!-- Scheme added for http -->
			/>
			<!-- etc... -->
		</sites>
  </sitecore>
</configuration>
```

Now the url is created correctly: `http://www.site1.com/page1/`. Yay!

Its not the perfect solution, setting the `scheme` in the config will override the `HttpContext.Current.Request.Url.Scheme` value, so if you are on `https` and the site is set to `http` the links will not be generated correctly. Because our requirement will only ever run on the authoring server, we have a transform for the delivery servers that removes the scheme from the site definition for the delivery servers.

###tl;dr
* Wrap the `LinkManager.GetItemUrl` in a `SiteContextSwitcher`
* In the SiteDefinition, Make sure the `targetHostName` and the `scheme` proeprties are set
* Enjoy urls with the host name when there is no `HttpContext`

-- Richard. 