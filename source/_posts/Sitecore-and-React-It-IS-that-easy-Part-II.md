---
title: Sitecore and React - It IS that easy - Part II
date: 2016-07-23 07:58:02
tags:
- Sitecore
- ReactJS
- Habitat
category: Sitecore.React
---
{% asset_img hero.jpg "Things are getting pretty serious!" %}

Well the last post was getting a bit long, so I decided to split it up! Please read it {% post_link Sitecore-and-React-It-IS-that-easy here %} if you haven't already.  

We left Sitecore with a new rendering type called `React Controller Rendering`, we have a `JsxResult`, a `JsxViewEngine` and finally a `JsxView` that takes our model from the controller action and passes it to [ReactJS.net](http://reactjs.net) to create the React component.

But we have a few questions hanging still. First we need to register our `jsx` file with ReactJS so it knows what the component is. Also we need to render the compiled JavaScript of the `jsx` file somewhere, otherwise this becomes just a server side component and we lose a lot of the fun bits with React.

### Lets use a pipeline!
As with a lot of things in Sitecore, pipelines and processors are great things to hook into. To register our `jsx` files we will use the `mvc.getPageRendering` pipeline. This pipeline is executed once per page, and in the args contains a list of all the renderings on the page.

```csharp
	public class AddJsxFIles : GetPageRenderingProcessor
	{
		public override void Process(GetPageRenderingArgs args)
		{
			this.AddRenderingAssets(args.PageContext.PageDefinition.Renderings);

			// Create the bundle for the render
			var bundle = new BabelBundle("~/bundles/react");

			foreach (var jsxFile in JsxRepository.Current.Items)
			{
				bundle.Include(jsxFile);

				if (!ReactSiteConfiguration.Configuration.Scripts.Any(s => s.Equals(jsxFile)))
				{
					ReactSiteConfiguration.Configuration.AddScript(jsxFile);
				}
			}

			BundleTable.Bundles.Add(bundle);
		}

		private void AddRenderingAssets(IEnumerable<Rendering> renderings)
		{
			foreach (var rendering in renderings)
			{
				var renderingItem = this.GetRenderingItem(rendering);
				if (renderingItem == null)
				{
					return;
				}

				if (renderingItem.TemplateID != Templates.JsxRendering.ID)
				{
					continue;
				}

				this.AddScriptAssetsFromRendering(renderingItem);
			}
		}

		private void AddScriptAssetsFromRendering(Item renderingItem)
		{
			var jsxFile = renderingItem[Templates.JsxRendering.Fields.JsxFile];
			if (!string.IsNullOrWhiteSpace(jsxFile))
			{
				JsxRepository.Current.AddScript(jsxFile, renderingItem.ID);
			}
		}

		private Item GetRenderingItem(Rendering rendering)
		{
			if (rendering.RenderingItem == null)
			{
				Log.Warn($"rendering.RenderingItem is null for {rendering.RenderingItemPath}", this);
				return null;
			}

			return rendering.RenderingItem.InnerItem;
		}
	}
```

So for each rendering, we check if it is a `Jsx Controller Rendering` and then we use the `Jsx File` field from that rendering to register the script with React. Notice too that we are building a bundle at the same time. This bundle will be used later. But we have to do both these things to run the code server side and client side.

### Bundles, Bundles, Bundles
The last step is to make sure that we include the bundled javascript in the main layout. Add this to your main layout towards the bottom.

```html
<script src="//fb.me/react-15.0.1.js"></script>
<script src="//fb.me/react-dom-15.0.1.js"></script>
@Scripts.Render("~/bundles/react")
```

### Putting it all together

Now we have all the component parts (pun intended!), we can create our first React Controller Rendering.  For this I chose the FAQ Accordion in the Habitat project as its a nice example of component parts. First I re-factored the view rendering into a new `React Controller Rendering`. The controller returns an `ActionResult`. We get the FAQ items from the repository and build the view model. Then we can return a new `JsxResult` - to simplify this process I added a an extension method to the `Controller` class to return the `JsxResult`

The controller action code is now:
```csharp
public ActionResult FaqAccordionReact()
{
    var renderingItem = RenderingContext.Current.Rendering.Item;

    if (!renderingItem?.IsDerived(Templates.FaqGroup.ID) ?? true)
    {
        return Context.PageMode.IsExperienceEditor ? this.InfoMessage(new InfoMessage(AlertTexts.InvalidDataSourceTemplateFriendlyMessage, InfoMessage.MessageType.Warning)) : null;
    }

    var model = this._faqRepository.GetFaqAccordion(renderingItem);

    return this.React("FaqAccordionReact", model);
} 
```

Now all we need is our `jsx` file that contains the React component:
```javascript
var FaqItem = React.createClass({
	render: function () {
		return (
		<div className="panel panel-default">
			<div className="panel-heading" role="tab" id="headingcollapse0" >
				<div className="panel-title">
					<a role="button" className="accordion-toggle" data-toggle="collapse" data-parent="#accordion" href={'#faq' + this.props.data.Id}>
						<span className="glyphicon glyphicon-search" aria-hidden="true"></span>
						<span dangerouslySetInnerHTML={{ __html: this.props.data.Question }} />
					</a>
				</div>
			</div>
			<div id={'faq' + this.props.data.Id} className="panel-collapse collapse" role="tabpanel" aria-labelledby="headingcollapse0">
				<div className="panel-body" dangerouslySetInnerHTML={{ __html: this.props.data.Answer }} />
			</div>
		</div>
		);
	}
});

var FaqAccordionReact = React.createClass({
	render: function () {
		return (
			<div className="panel-group" id="accordion" role="tablist" aria-multiselectable="true">
				{this.props.data.Items.map(function(faq) {
					return <FaqItem key={faq.Id} data={faq}/>;
				})}
			</div>
		);
	}
});
```

### Notes on the Model
Note that we are setting the fields by using `dangerouslySetInnerHTML` - this allows us to support the page editor when rendering the fields. Also note that when creating the model, the fields need to be rendered into a string property that we can pass through to the React component.

This is the code to build the FaqItems:

```csharp
faqItems.Add(new FaqItem
{
    Id = item.ID.ToShortID().ToString(),
    Question = FieldRenderer.Render(item, Templates.Faq.Fields.Question.ToString()),
    Answer = FieldRenderer.Render(item, Templates.Faq.Fields.Answer.ToString())
});
```

### In Action:
It's hard to get everything here in the post, so please checkout my [fork of Habitat](https://github.com/GuitarRich/Habitat) with this integration enabled. Specifically the [React Foundation Layer](https://github.com/GuitarRich/Habitat/tree/master/src/Foundation/React), and the [modified FAQ Accordion rendering](https://github.com/GuitarRich/Habitat/tree/master/src/Feature/faq/code). Just follow the normal steps to install Habitat and you should be able to see the accordion rendering at [http://habitat.dev.local/en/Modules/Feature/FAQ](http://habitat.dev.local/en/Modules/Feature/FAQ).

Here is the rendering running with dev tools open so you can see the react markup:
{% asset_img faqmodule.png "It IS that easy!" %}


## Tada!! and Next Steps
So far we have had some excellent results from the renderings we have created with React. Next steps we are going to look at the performance of the react renderings and how they cope with caching. Also I want to do a bit more to enable dynamic placeholders.

Well thats all for now - I'm sure I didn't cover everything, so please ask any questions in the comments or on the [Sitecore slack community](http://sitecorechat.slack.com).

#### Credits
---
- Alex Shyba & Jacob Christensen for previous posts and inspiration
- Justin Laster at [Lightmaker](http://lightmaker.com) for collaborating and putting up with constant questions on what might be the best way of doing this!
