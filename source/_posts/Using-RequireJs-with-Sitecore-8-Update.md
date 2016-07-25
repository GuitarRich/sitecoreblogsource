title: "Using RequireJs with Sitecore 8 - Update"
date: 2015-10-19 13:37:58
tags:
- Sitecore
- JavaScript
- RequireJS
category: Sitecore
---

This update has been a long time in coming! Last year I blogged about using [Require.js](http://requirejs.org) to [organize your Sitecore JavaScript](http://www.sitecorenutsbolts.net/2014/11/03/Using-Require-to-Organize-Your-Sitecore-Javascript/). In the post we looked at setting up require js with [jQuery](https://jquery.com/). This was the `main.js` config file:

```
require.config({
	baseUrl: "/assets/js",
	paths: {
		jquery: "vendor/jquery.min",
		"jquery.migrate": "vendor/migrate",
		"jquery.mobile.events": "vendor/mobile.events",
		"jquery.royalslider": "vendor/jquery.royalslider",
	},
	shim: {
		"jquery.migrate": {
			deps: ["jquery"]
		},
		"jquery.mobile.events": {
			deps: ["jquery"]
		},
		"jquery.royalslider": {
			deps: ["jquery"]
		},
	}
});

// Now run any initialization javascript for the site:
require(["jquery"], function($) {
	// Init code here
});
```

This worked great in Sitecore 7.x, but in Sitecore 8 the page editor started breaking, lots of JavaScript errors. The reason for this is that the **Prototype** JavaScript library was used by the Experience Editor and conflicts with **jQuery**. [See this KB Article](https://kb.sitecore.net/articles/286042)

The obvious answer is to run jQuery in **no-conflict** mode. But how does that work with RequireJs?

One option would be to take jQuery out of the require dependencies and load it at the top of the page, put it in no-conflict mode and use an [IIFE](http://benalman.com/news/2010/11/immediately-invoked-function-expression/) to load jQuery into to the module:

```
(function($) {
    // plugin load code here
})(jQuery);
```
Credits: [Sitecore preview mode and loading jQuery](http://kamsar.net/index.php/2013/10/sitecore-preview-mode-and-loading-jquery/)

But that kinda goes against the benefits and reasons for using RequireJs in the first place. Fortunately it is pretty easy to run jQuery in no-conflict mode using Require.

###jQuery as an AMD Module
The first thing we need to do is define a new AMD module that puts jQuery into no conflict mode:

####jquery.no-conflict.js
```
define(["jquery"], function(jq) {
  return jq.noConflict(false);
});
```

Now we can add that module to our requirejs paths and then map it to our `jquery` path that we setup earlier in the post:

```
require.config({
	baseUrl: "/assets/js",
	paths: {
		jquery: "vendor/jquery.min",
		"jquery-private": "modules/jquery.no-conflict",
		"jquery.migrate": "vendor/migrate",
		"jquery.mobile.events": "vendor/mobile.events",
		"jquery.royalslider": "vendor/jquery.royalslider",
	},
	shim: {
		"jquery.migrate": {
			deps: ["jquery"]
		},
		"jquery.mobile.events": {
			deps: ["jquery"]
		},
		"jquery.royalslider": {
			deps: ["jquery"]
		},
	},
	map: {
		// '*' means all modules will get 'jquery-private'
		// for their 'jquery' dependency.
		'*': {
			'jquery': 'jquery-private'
		},
			// 'jquery-private' wants the real jQuery module
			// though. If this line was not here, there would
			// be an unresolvable cyclic dependency.
			'jquery-private': {
			'jquery': 'jquery'
		}
	}
});

// Now run any initialization javascript for the site:
require(["jquery"], function($) {
	// Init code here
});
```

Notice that everything else still references the `jquery` path. Its the map that does the magic here!

So now you can still use jQuery and the Experience Editor without going and changing all your existing AMD Modules!

Happy coding!

--Richard

References: [Mapping Modules to Use NoConflict](http://requirejs.org/docs/jquery.html#noconflictmap)