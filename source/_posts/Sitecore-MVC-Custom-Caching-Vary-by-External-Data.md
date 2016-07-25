title: "Sitecore MVC Custom Caching - Vary by External Data"
date: 2014-12-02 09:36:01
tags:
- Sitecore
- MVC
- Caching
- Cache
- Pipelines
category:
- Sitecore
---

Sitecore’s caching system is very powerful and when tuned right can help to create an implementation that performs well under load, which always keeps the clients happy! There are a number of great blog posts that deal with how the Sitecore caching systems work along with the SDN documentation so I wont go into great detail on those. (Links below).

Today we will just be looking at the HTML Cache. This caches the actual HTML generated from renderings or sublayouts. The cache is configured per rendering or sublayout on the presentation controls. The control can be set to always cache, or you can set it to vary the cache on different settings. For example, you can set the control to “Vary by Data” – this will cache the html uniquely dependent on the Datasource or Context.Item for the control. 

The options for varying the cache are:
*   By Data
*   By Device
*   By Login
*   By Parameters
*   By Query String
*   By User

{% asset_img Presentation.jpg "Setting the caching values" %}

This is a really important cache to get right. It can help with slower renderings or sublayouts. A common example is a Navigation rendering, these are normally common on every page of the site and can require a lot of items to be accessed, especially if we are dealing with a meganav style. Caching this rendering means that the navigation would only be slow if the cache is empty, all subsequent requests would just render the cached html and improve performance.

One thing to note is that any time a publish operation completes, the HTML Cache gets completely cleared out!

## Creating a Custom Vary By

### Scenario

At Lightmaker we have a number of sites that include external data alongside the standard Sitecore content. To get the data we use wildcards to generate dynamic Urls that allow us to know which data to get. An example of this would be on a sports based website. The site needs to display information about the players of the sport, but the player information is already in a back end system that the client will still use. Rather than duplicate the data in the Sitecore content tree, a web service was provided to access the player information, this service returns us a model of the player data in json.

The url for player pages should look like these:

*   http://mysportssite.com/players/player-name-1/overview
*   http://mysportssite.com/players/player-name-2/overview
*   http://mysportssite.com/players/player-name-3/overview

Our Sitecore content tree would be:

{% asset_img Sitecore-Tree.jpg "Sitecore Tree" %}

### Resolving the Player

To resolve the player we add an extra processor to the **httpRequestBegin** pipeline. In that we check the Context.Item template to make sure we are on the **Player Overview** item, then parse the url for the player name. We can then call the 3<sup>rd</sup> party web service to get the player information and build a **Player** object from the json model.

Once we have that we add it to the **Sitecore.Context.Items** dictionary. Resolving the player this way allows us to reference the player data from multiple renderings on the page without having to request the player each time.

This does give us a problem tho, we can no longer cache the rendering as none of the Vary By options apply. Each url is resolving to the same Sitecore item and the same renderings on that item, so we need to vary by the 3<sup>rd</sup> party data.

### Custom Cache Key Generator

The solution to this is to override the **GenerateCacheKey** processor. The **Vary By** options change the way the cache key is generated, so we can just add an extra part to the key that defines the player from the 3<sup>rd</sup> party service.

Code:

``` csharp
namespace LM.Ignite.Mvc.Pipelines.Response
{
	using System.Linq;

	using LPGA.Model.Templates.Ignite;

	using Sitecore;
	using Sitecore.Data.Items;
	using Sitecore.Mvc.Pipelines.Response.RenderRendering;
	using Sitecore.Mvc.Presentation;

	public class GenerateCacheKey : Sitecore.Mvc.Pipelines.Response.RenderRendering.GenerateCacheKey
	{
		protected override string GenerateKey(Rendering rendering, RenderRenderingArgs args)
		{
			var cacheKey = base.GenerateKey(rendering, args);
			var caching = rendering.Caching;

			if (caching.VaryByData)
			{
				// Add in the player item to the cache key so we can vary by tournament/player
				var player = Context.Items["ContextPlayer"] as Item;
				if (player != null)
				{
					cacheKey += "_#player:" + player.ID.ToShortID();
				}
			}

			return cacheKey;
		}
	}
}
```

To add this in we can patch the Sitecore config via an include file:

``` xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<pipelines>
			<mvc.renderRendering>
				<processor patch:instead="processor[@type='Sitecore.Mvc.Pipelines.Response.RenderRendering.GenerateCacheKey, Sitecore.Mvc']"
					 type="LM.Ignite.Mvc.Pipelines.Response.GenerateCacheKey, LM.Ignite.Core" />
			</mvc.renderRendering>
		</pipelines>
	</sitecore>
</configuration>
```

For this example, I have just used the **Vary By Data** as my option to enable the custom cache key generation. This was a good fit for our project as it was the data for the rendering that changed.

### Web Forms

This example only applies if you are building an Mvc solution with Sitecore. WebForms handle the cache key generation slightly differently. The cache key is generated as part of the **Sitecore.Web.UI.WebControl** base class, so you could implement a similar solution by creating a custom web control base class and having your SubLayouts inherit from that.

Now we can safely bring in 3<sup>rd</sup> party data and still benefit from the Sitecore HTML Cache without resorting to a complicated custom caching solution.

– Richard Seal

References for Sitecore Caching:

*   [http://learnsitecore.cmsuniverse.net/en/Developers/Articles/2009/07/CachingOverview.aspx](http://learnsitecore.cmsuniverse.net/en/Developers/Articles/2009/07/CachingOverview.aspx)
*   [http://sitecoreblog.patelyogesh.in/2013/06/how-sitecore-caching-work.html](http://sitecoreblog.patelyogesh.in/2013/06/how-sitecore-caching-work.html)