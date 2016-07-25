title: "Using Require to Organize Your Sitecore Javascript"
date: 2014-11-03 09:35:11
tags:
- Sitecore
- JavaScript
- requirejs
category: Sitecore
---
One of the problems when implementing a Sitecore website is how to include your JavaScript files. Each rendering/sublayout has the potential for custom JavaScript to enable the functionality for that component.

### Options
So what are the options?

1. We could just add all our JavaScript files as links in the head tag of the Main Layout for the site.

	While this would make sure that all the JavaScript is loaded and ready for the components to use, we know that loading a lot of JavaScript in the head tag is bad practice, it blocks the rest of the html on the page and could cause slow rendering times for the user.

2. We could add the JavaScript files in at the bottom of the page in the Main Layout for the site. 
	
	This follows JavaScript best practice, our script now does not block the page load, but it does leave us with another problem. Because we don’t know which modules will be loaded on the page, we have difficulty initializing our components. We could check for each elements existence on the page, but this would lead to a lot of DOM checking and again potentially slow down the rendering of our page.

3. We could customize the rendering template to allow us to define which JavaScript files should be loaded for the rendering, and write a custom pipeline to make sure they are loaded. 

	This is getting better, but we would still have to load the JavaScript in the head tag to make sure it is available.

### RequireJS to the Rescue

Fortunately there is another solution – in many large sites [RequireJs](http://requirejs.org/ "RequireJS") can be used to help organize and load only those scripts that are required for the components on the page. Each component can require the JavaScript modules needed and RequireJS handles loading those files asynchronously, no blocking. So how can we use that with Sitecore?

#### Setup

These steps assume some knowledge of RequireJS, if you haven’t used it before I highly recommend looking through the require site. [http://requirejs.org](http://requirejs.org/ "RequireJS")  

First we need to set our MainLayout and RequireJS scaffolding up. The following code assumes your JavaScript is located at __"/assets/js"__ and organized into logical folders. The main JavaScript file sets up the require config and any initial setup:

main.js:
``` js
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

Add the following script tags to your Main Layout in the head tags. This is ok as RequireJS loads in the JavaScript files async, so it doesn’t block the page.

``` html
<script src="/assets/js/vendor/require.js"></script>
<script src="/assets/js/main.js"></script>
```

#### Components

Now we have the setup we can write the JavaScript for our components as RequireJS modules and require them directly from our renderings/sublayouts

Example Require Module (Located at: __/assets/js/modules/slider.js__)

``` js
define(["jquery", "jquery.royalslider", "jquery.mobile.events"], function($) {
  var BasicSlider;
  return BasicSlider = (function() {
    function BasicSlider(options) {
      this.options = options;
      this.options = $.extend({}, this.defaults, this.options);
      this.initSlider();
    }

    DetailCarousel.prototype.defaults = {
      element: "#basicSlider",
    };

    DetailCarousel.prototype.initSlider = function() {
		// TODO: Initialise the slider here
    };

    return DetailCarousel;
  })();
});
```

Example Rendering (Razor Script)
``` html
<div class="slider"><!-- Slider Html Structure Here --></div>
<pre>
<script>
require(["jquery", "modules/slider"], function($, Slider){
    new Slider({
        element: '.slider'
    });
});
</script>
```

#### Conclusion

Using this method, we can write reusable RequireJS modules for our JavaScript and include only what is required for the page without any page blocking and any hacky work arounds to get the components working.

Another nice side effect of this, is that it all works nicely with the page editor. Because of the way RequireJS loads the JavaScript in and then executes the function, when components are added via the page editor, the script is executed as normal. Of course you can turn this off if you don’t want the JavaScript to execute in page edit mode.

– Richard Seal