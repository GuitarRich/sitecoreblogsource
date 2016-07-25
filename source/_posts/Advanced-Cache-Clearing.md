title: "Advanced Cache Clearing"
date: 2016-04-26 14:02:18
tags:
category: Sitecore
---

A recent post on [Stack Overflow](http://stackoverflow.com/questions/36796980/sitecore-programmatically-clear-users-cache) asked about clearing out the `HtmlCache` for a specific user, the short answer is in the post. But I thought I'd add some more detail to that answer here for anyone else wants to know the Sitecore Html Cache a little more in depth.

###What's the HtmlCache??
The `HtmlCache` is the top level cache, sometimes call the web cache. It is used to store the rendered Html from your renderings and sublayouts if you are still using webforms. This cache can be configured on a per-site basis.

{% asset_img cacheoverview.jpg "Caching Overview" %}
Credit: [Learn Sitecore: Caching Overview](http://learnsitecore.cmsuniverse.net/Developers/Articles/2009/07/CachingOverview.aspx)

You can enable the Html cache either on your rendering or sublayout item, or you can specify on each instance of a rendering in presentation. Each method includes options to vary the cached output based on defined parameters:

{% asset_img cachevaryby.gif "Cache Vary By Parameters" %}

###Why use it?
The simple answer is... performance! The Html cache is often the last consideration when imnplementing a Sitecore site. I feel that it should be one of the first. It really should be part of the overall plan of the site, because when used right it can provide massive performance gains.

#####Caching is not an excuse for slow code!
Don't get the wrong idea, if you are using caching to band aid over badly performing code, you are going down a bad path!

But using the html cache on already well performing renderings can really help with how many requests per second the server can cope with.

###Generating the Cache Key
So how do all those parameters work? Well, we are going to deal with an MVC implementation as its a lot nicer to deal with, but the principles are the same, the code is just tightly hooked into the base web control class.

All those options on the vary by parameters simply change the way the Cache Key is built for the rendering. This is done via the `Sitecore.Mvc.Pipelines.Response.RenderRendering.GenerateCacheKey` processor that runs in the `mvc.RenderRendering` pipeline.

Every cache key starts by adding the unique key for the current rendering. That consists of a type and then some property values. For example a controller rendering key is built like this:
```csharp
return "controller::" + this.ControllerName + "#" + this.ActionName;
```

A view rendering is:
```csharp
return "view::" + this.ViewPath;
```

The sites current context language is also added into the Key

#####Careful when using MVC Area's
You may notice that in the controller rendering, the `Area` field is not used anywhere to build the cache key. This means that if you have 2 controllers of the same name but in different area's, you need to make sure that the action names are unique still. If not you will get contamination when the renderings output is cached.

Most of the time this will not be a problem, because its rare to not select any of the other vary by parameters, but it is something to keep in mind when designing your rendering hierarchy.

####Vary by:
* Data: This adds the rendering items path to the key: `"_#data:" + rendering.Item.Paths.Path;`
* Device: This adds the current `Context.GetDeviceName` to the key: `"_#dev:" + Context.GetDeviceName();`
* Login: This adds a boolean to the key that specifies if the user is logged in or out: `"_#login:" + (object) Context.IsLoggedIn;`
* Parameters: This concatenates all the rendering parameters to a query string and puts them in the key: `"_#parm:" + rendering.Parameters.ToQueryString();`
* Query String: Does what it says on the tin! Adds the current request query string to the key: `"_#qs:" + MainUtil.ConvertToString(request.QueryString, "=", "&");`
* User: This appends the current Context user to the key. Note that if the user is not logged in, this will be `extranet/anonymous` when using a default site definition: `"_#user:" + Context.GetUserName();`

####Fun With Cache Keys
What this gives us is a really simple way to make sure that our cache key is generated with the right variation to give a nice performance boost on the site, but still provide relavent information to the user.

A while ago I wrote a post on [caching a rendering using external data](http://www.sitecorenutsbolts.net/2014/12/02/Sitecore-MVC-Custom-Caching-Vary-by-External-Data/) - this showed how to add an element from some external data into the cache key. This might be useful when displaying dynamic pages of content that include external feeds or other non-Sitecore based content. For example a rendering that displayed an adverted served by an external ad server.

Another scenario we ran into recently was a site that had 3 states for a user.
* Anonymous
* Registered
* Subscribed

In this case, you could register on the site and get free content, or subscribe and get premium content. For this we created a new cache key part that held the status for the user, this replaced the boolean for **Vary By Logged In** part.

###Advanced Cache Clearing
By default, the Html cache is cleared after a publish. On the `publish:end` and `publish:end:remote` events, the config sets up the HtmlCache clearer:

```xml
<event name="publish:end">
    <handler type="Sitecore.Publishing.HtmlCacheClearer, Sitecore.Kernel" method="ClearCache">
        <sites hint="list">
            <site>website</site>
        </sites>
    </handler>
</event>
<event name="publish:end:remote">
    <handler type="Sitecore.Publishing.HtmlCacheClearer, Sitecore.Kernel" method="ClearCache">
        <sites hint="list">
            <site>website</site>
        </sites>
    </handler>
</event>
```

By default this cache clears right away on the event, but you can add an attribute to the Site Definition called `htmlCacheClearLatency` - this is a time span that specifies how long after the event is raised you want the cache to be cleared. This could be useful if you want to wait for indexes to be updated before clearing the cache.

But we can be a bit more refined than that!

The `HtmlCache` class inherits from `Sitecore.Caching.CustomCache` - so while we can clear all and remove by a certain key, there is an interesting litte method called `RemoveKeysContaining(string value)` and `RemovePrefix(true)`.

Both of these methods can be used to selectively remove entries from the cache. In the Stack Overflow question at the top of the post, the developer wanted to remove a specified user from the cache.  If the vary by parameters have been set correctly, this becomes a simple task. 

Lets say we have a rendering called `User Profile`, we set the caching up like this:

{% asset_img userprofilecache.gif "User Profile Vary By Parameters" %}

The rendering is a controller rendering setup like this:

{% asset_img userprofilerendering.png "User Profile rendering" %}

For my login, the cache key would be:
```
"controller::Users#Profile_#lang:en_#login:True_#user:extranet/richardseal"
```

When I update my profile by a form submission, I want to clear this renderings cache, and any other renderings that might use part of my profile. This is nice and simple:

```csharp
// Need to clear the cache for the header and the user profile....
var htmlCache = CacheManager.GetHtmlCache(Context.Site);

// Remove all cache keys that contain the currently logged in user.
var cacheKeyPart = $"_#login:True_#user:{Context.GetUserName()}";
htmlCache.RemoveKeysContaining(cacheKeyPart);
```

What about clearing all cached renderings that use a specific item as thier datasource?
```
// Need to clear the cache for the current context item
var htmlCache = CacheManager.GetHtmlCache(Context.Site);
var path = Sitecore.Context.Item.Paths.Path;

// Remove all cache keys that contain the currently logged in user.
var cacheKeyPart = $"_#data:{path}";
htmlCache.RemoveKeysContaining(cacheKeyPart);
```

The possibilities are endless. So have fun with the HtmlCache - hopefully this will be of use to someone :)

-- Richard
