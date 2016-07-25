---
title: Sitecore and React - It IS that easy!
date: 2016-07-23 07:58:02
tags:
- Sitecore
- ReactJS
- Habitat
category: Sitecore
---
{% asset_img hero.jpg "Richard was almost ready to implement his Hello World React app" %}

At [Lightmaker](http://lightmaker.com) we have had a number of discussions recently about how to streamline the front end build into a Sitecore environment. Our front end devs are all Mac based, so currently can't run a local instance of Sitecore. TBH - it would be overkill for them to do that anyway. But how can they accurately test their code before it gets into a Sitecore rendering? Also how can we remove the process where front end make a fix and that fix has to then be applied to a razor view etc... The front end team should just be able to update markup and not have to worry about then integrating that back into a Sitecore rendering - warning - this may be a long post!

##### ReactJS - Is it an option?
---
Various ideas were discussed, [Angular](https://angularjs.org/), a [Jade](http://jade-lang.com/) view engine etc... Ultimately we looked to [React](https://facebook.github.io/react/) as a tool to build the front end code with. 

Why?
* Component-Based: Just like Sitecore React is component based, we can easily take a React component and match that to a rendering we would have built in Sitecore
* Front End Development doesn't need to worry about Sitecore, ASP.NET MVC or Razor script. As long as they know React, they can build Sitecore components
* React is very efficient with DOM updates. React calculates the changes that need to be made in the DOM before making them and updates the DOM tree, avoiding costly DOM operations
* We can render the components on the server, making it great for SEO

At the same time we have been looking into this, some other blog posts have popped up with React+Sitecore ideas. These were an inspiration for our final solution: [Sitecore and React: How Hard Can it Be](http://sitecoreblog.alexshyba.com/sitecore-and-react-how-hard-can-it-be/) showed a nice way that a react component could be built with WebForms, that takes the renderings datasource and adds it to the React model. But I didn't want to use WebForms. On to [Sitecore.Pathfinder](https://github.com/JakobChristensen/Sitecore.Pathfinder/tree/master/src/Features/Sitecore.Pathfinder.React), Here Jakob had taken what Alex did with WebForms and build a `JsxRenderer` - now we are not tied down to WebForms, which is great. But both of these solutions only cope with very simple renderings that only read data from a single Item without any manipulation before rendering it.

### What are we trying to Achieve?
So what are our goals for this?
* Experience Editor must still work
* We must be able to build the data model as needed from the code
* FED should be able to develop components independently of Sitecore 
* It should be easy to create a new React component in Sitecore. No jumping through hoops if we can help it!

This post will outline the method that Justin Laster @ [Lightmaker](http://lightmaker.com) and I came up with to use React with Sitecore. Note that our solution has been inspired by others and these will be credited throughout the post.

### Controller Rendering - Not a great option
One option that we looked at was to use a `Controller Rendering` and then in the razor view use [ReactJS.net](http://reactjs.net/) to generate the React component. This worked, but had some major drawbacks.

First every React rendering would need 2 files creating. The `jsx` file and a `cshtml` file. It would look something like this:

* Helloworld.jsx
```javascript
var HelloWorld = React.createClass({
  render: function() {
    return <div>Hello world!</div>;
  }
});
```

* Helloworld.cshtml
```html
@model MyComponent.HelloWorldModel

@Html.React("HelloWorld", Model)
```

On top of that, we need to register the jsx file with ReactJS and add it to a bundle. So multiple places that we have to set this up make it a bit of a maintenance headache!

### JsxViewEngine - Lets get serious
Lets crack this baby open! Here is the github repo for the React/Sitecore module can be found [here](https://github.com/GuitarRich/Habitat) - I have integrated this into the [Habitat](https://github.com/Sitecore/Habitat) project so we have a site to work with.

First lets start by creating a new `React Controller Rendering` - we are creating this because we need to add a couple of fields that are not on a standard controller rendering:

{% asset_img JsxControllerRendering.png "Task Runner Explorer" %}

The new fields are:
- `Jsx File`: Pretty self explanatory, but this specifies the location of the Jsx file that contains our React component. While we can work this out from the controller action. This gives more flexibility if we want to include multiple components in a single Jsx file.
- `Placeholders`: This lists any placeholders that are present in our component. We need this here as currently we are not parsing the Jsx code directly to get the placeholders. This is on the roadmap to make easier.

Of course, we can't forget to set a snappy icon - [@jammykam](https://twitter.com/jammykam) would not be happy!

At this point we are still using a Controller Action to render our component. So we need to make that action return a `JsxResult` rather than a `ViewResult`.  

```csharp
public class JsxResult : ViewResult
{
}
```

Wait - where is the code!! For the `JsxResult` class we are just inheriting from the standard Mvc `ViewResult`. We don't need to change any of the implementation here, but we have the class ready for future updates.

Next we need a view engine. The view engine tells us where to look for the jsx files:

```csharp
	public class JsxViewEngine : BuildManagerViewEngine
	{
		/// <summary>Initializes a new instance of the <see cref="T:System.Web.Mvc.RazorViewEngine" /> class.</summary>
		public JsxViewEngine()
			: this(null)
		{
		}

		/// <summary>Initializes a new instance of the <see cref="T:System.Web.Mvc.RazorViewEngine" /> 
        /// class using the view page activator.</summary>
		/// <param name="viewPageActivator">The view page activator.</param>
		public JsxViewEngine(IViewPageActivator viewPageActivator)
			: base(viewPageActivator)
		{
			this.AreaViewLocationFormats = new []
			{
				"~/Areas/{2}/Views/{1}/{0}.jsx",
				"~/Areas/{2}/Views/Shared/{0}.jsx"
			};
			this.AreaMasterLocationFormats = new []
			{
				"~/Areas/{2}/Views/{1}/{0}.jsx",
				"~/Areas/{2}/Views/Shared/{0}.jsx"
			};
			this.AreaPartialViewLocationFormats = new []
			{
				"~/Areas/{2}/Views/{1}/{0}.jsx",
				"~/Areas/{2}/Views/Shared/{0}.jsx"
			};
			this.ViewLocationFormats = new []
			{
				"~/Views/{1}/{0}.jsx",
				"~/Views/Shared/{0}.jsx"
			};
			this.MasterLocationFormats = new []
			{
				"~/Views/{1}/{0}.jsx",
				"~/Views/Shared/{0}.jsx"
			};
			this.PartialViewLocationFormats = new []
			{
				"~/Views/{1}/{0}.jsx",
				"~/Views/Shared/{0}.jsx"
			};
			this.FileExtensions = new []
			{
				"jsx"
			};
		}

		/// <summary>Creates a partial view using the specified controller 
        /// context and partial path.</summary>
		/// <returns>The partial view.</returns>
		/// <param name="controllerContext">The controller context.</param>
		/// <param name="partialPath">The path to the partial view.</param>
		protected override IView CreatePartialView(ControllerContext controllerContext, string partialPath)
		{
			return new JsxView(controllerContext, partialPath, null, false, this.FileExtensions, this.ViewPageActivator)
			{
				DisplayModeProvider = this.DisplayModeProvider
			};
		}

		/// <summary>Creates a view by using the specified controller context and the 
        /// paths of the view and master view.</summary>
		/// <returns>The view.</returns>
		/// <param name="controllerContext">The controller context.</param>
		/// <param name="viewPath">The path to the view.</param>
		/// <param name="masterPath">The path to the master view.</param>
		protected override IView CreateView(ControllerContext controllerContext, string viewPath, string masterPath)
		{
			return new JsxView(controllerContext, viewPath, masterPath, true, this.FileExtensions, this.ViewPageActivator)
			{
				DisplayModeProvider = this.DisplayModeProvider
			};
		}
	}
```

Again fairly standard stuff here. Its worth noting the paths for the `.jsx` files. They are setup to be in the same place as you would put your `.cshtml` files. This means we can still keep all our component parts together in the same project. We could customize this to be whatever folder structure fits the project, but this seemed like a good fit.

We need to make sure that the view engine is registered with asp.net. We can do this by adding an `App_Start` class and calling `ViewEngines.Engines.Add(new JsxViewEngine());`

Also notice that the `CreateView` and `CreatePartialView` methods are returning a `JsxView` object. Lets see what is in there:
```csharp
	public class JsxView : BuildManagerCompiledView
	{
        /*
         * Properties and constructors removed for brevity
         */

        internal Sitecore.Mvc.Presentation.Rendering Rendering => RenderingContext.Current.Rendering;

		/// <summary>Renders the specified view context by using the specified writer and Jsx component.</summary>
		/// <param name="viewContext">The view context.</param>
		/// <param name="writer">The writer that is used to render the view to the response.</param>
		/// <param name="instance">Not used in this view engine</param>
		protected override void RenderView(ViewContext viewContext, TextWriter writer, object instance)
		{
			if (writer == null)
			{
				throw new ArgumentNullException(nameof(writer));
			}

			var componentName = Path.GetFileNameWithoutExtension(this.ViewPath)?.Replace("-", string.Empty);
			var props = this.GetProps(viewContext.ViewData.Model);

			IReactComponent reactComponent = this.Environment.CreateComponent(componentName, props);
			writer.WriteLine(reactComponent.RenderHtml());
		}

		private IReactEnvironment Environment
		{
			get
			{
				try
				{
					return ReactEnvironment.Current;
				}
				catch (TinyIoCResolutionException ex)
				{
					throw new ReactNotInitialisedException("ReactJS.NET has not been initialised correctly.", ex);
				}
			}
		}

		protected virtual dynamic GetProps(object viewModel)
		{
			dynamic props = new ExpandoObject();
			var propsDictionary = (IDictionary<string, object>)props;

			dynamic placeholders = new ExpandoObject();
			var placeholdersDictionary = (IDictionary<string, object>)placeholders;

			propsDictionary["placeholders"] = placeholders;
			propsDictionary["data"] = viewModel;

			var placeholdersField = this.Rendering.RenderingItem.InnerItem["Place Holders"];
			if (string.IsNullOrEmpty(placeholdersField))
			{
				return props;
			}

			var controlId = this.Rendering.Parameters["id"] ?? string.Empty;
			dynamic placeholderId = null;

			var placeholderKeys = placeholdersField.Split(Constants.Comma, StringSplitOptions.RemoveEmptyEntries).Select(p => p.Trim()).ToList();
			foreach (var placeholderKey in placeholderKeys)
			{
				if (placeholderKey.StartsWith("$Id."))
				{
					if (placeholderId == null)
					{
						placeholderId = new ExpandoObject();
						placeholdersDictionary["$Id"] = placeholderId;
					}

					((IDictionary<string, object>)placeholderId)[placeholderKey.Mid(3)] = PageContext.Current.HtmlHelper.Sitecore().Placeholder(controlId + placeholderKey.Mid(3)).ToString();
				}
				else
				{
					placeholdersDictionary[placeholderKey] = PageContext.Current.HtmlHelper.Sitecore().Placeholder(placeholderKey).ToString();
				}
			}

			return props;
		}
	}
```

Ok - now we are getting to the bit that does the work. Ultimately its a pretty simple bit of code. First we get the component name from the ViewPath: `var componentName = Path.GetFileNameWithoutExtension(this.ViewPath)?.Replace("-", string.Empty);`. Then we can build the data model to be serialized and sent to the React component: `var props = this.GetProps(viewContext.ViewData.Model);`.

`GetProps` builds us a dynamic object with 2 root properties, `data` and `placeholders`. The `data` property will hold all the properties on the view model. Note that our view model **must** be serializable. The `placeholders` property holds the rendered contents of that placeholder. The contents of each placeholder is then added via the React code.

Finally we create the React component using ReactJS: `IReactComponent reactComponent = this.Environment.CreateComponent(componentName, props);` and then write that out to the `TextWriter`: `writer.WriteLine(reactComponent.RenderHtml());`. 

You may be wondering tho - how does that work? Don't you need to render the JavaScript somewhere, or register the `jsx` file with ReactJS? Well yes, we do. This is where we have taken a slightly different approach to previous posts. And it will all be dealt with in {% post_link Sitecore-and-React-It-IS-that-easy-Part-II "Part II here" %}