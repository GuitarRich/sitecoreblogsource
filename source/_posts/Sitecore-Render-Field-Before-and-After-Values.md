title: "Sitecore Render Field - Before and After Values Fixed"
date: 2015-10-26 12:23:48
tags:
- Sitecore
- Render Field
- Pipelines
category: Sitecore
---

I have come across this issue many times in implementing Sitecore websites. You need a field to show on the page with some surrounding markup. The field is optional, and if the field has no content, the markup should not be displayed at all. A common way I have seen this done is by just checking the value of the field before calling the render method:

```html
@if (!string.IsNullOrWhiteSpace(item["myfield"])){
	<div class="surrounding-markup-styles">
		@Html.Sitecore().Field("myfield", item)
	</div>
}
```

But apart from looking a bit ugly and verbose, it means we have to get the contents of the field twice. We could do that in the controller/business layer of our project. But then we would be mixing presentation into the business layer. Also in this example, the experience editor is not supported. We would have to add an extra check for page edit mode!

```html
@if (!string.IsNullOrWhiteSpace(item["myfield"]) || Sitecore.Context.PageMode.IsPageEditor){
	<div class="surrounding-markup-styles">
		@Html.Sitecore().Field("myfield", item)
	</div>
}
```

No imagine that you have 10 fields like that on your rendering, things are starting to get very ugly!

### RenderFieldArgs
In the `RenderFieldArgs` we find a couple of properties that might help. There is `EnclosingTag`, this is just a string containing the type of tag, `h1`, `div` etc... - the field renderer pipeline then surrounds the field content with that tag. Helpfull, but a bit limiting.

There are also properties for `Before` and `After` - now we are getting closer. These properties take a string containing whatever markup you want to add and logically add those string *before* and *after* the field.

So how do we use those parameters. Well one of the overloads for `Html.Sitecore().Field` is `string fieldName, Item item, object parameters` - this parameters object takes a dynamic object and converts the properties on the dynamic to matching properties on `RenderFieldArgs`, so using this method you can set *any* property in the `RenderFieldArgs` object.


Example (Standard Sitecore):
```html
<!-- Enclosing Tag - renders "<h2>Field Content</h2>" -->
@Html.Sitecore().Field("myfield", item, new { EnclosingTag = "h2" })

<!-- Before & After - renders "<div class="example">Field Content</div>" -->
@Html.Sitecore().Field("myfield", item, new { Before = "<div class=\"example\">", After = "</div>" })

```

As a site note, if you are using [Fortis](http://fortis.ws) - we have mirrored this implementation of the field renderer. So you can use the `.Render` method of a field wrapper and set the same properties.

Example (Using Fortis):
```html
<!-- Enclosing Tag - renders "<h2>Field Content</h2>" -->
@Model.MyField.Render(new { EnclosingTag = "h2" })

<!-- Before & After - renders "<div class="example">Field Content</div>" -->
@Model.MyField.Render(new { Before = "<div class=\"example\">", After = "</div>" })
```

This looks much better. Nice and clean and keeps page editor support. You can imagine my dissapointment then when I discovered that Sitecore renders both the enclosing tag and the before and after fields, even if the field content is empty. 

> ####UPDATE: 
> The enclosing tag field does not render the tags if the field is empty, and so works as expected. Before and After do not work the same way.

### Time to override the Processor!

{% asset_img all-the-processors.jpg "Override All The Processors!" %}

The pipeline pipeline that handles all this is the imaginatively titled `AddBeforeAndAfterValues` in the `RenderFieldPipeline`. Its a pretty simple processor and the change to make it work as expected was also fairly simple:

```csharp
using Sitecore.Diagnostics;
using Sitecore.Pipelines.RenderField;
using Sitecore.StringExtensions;

/// <summary>
/// Adds the before, after and enclosing tag.
/// 
/// </summary>
public class AddBeforeAndAfterValues : Sitecore.Pipelines.RenderField.AddBeforeAndAfterValues
{
	/// <summary>
	/// Runs the processor.
	/// 
	/// </summary>
	/// <param name="args">The arguments.</param>
	public new void Process(RenderFieldArgs args)
	{
		Assert.ArgumentNotNull(args, "args");

		// Don't add any extra markup if the field is empty, unless we are in page edit mode
		if (Sitecore.Context.PageMode.IsPageEditor == false && string.IsNullOrWhiteSpace(args.Result.FirstPart) &&
			string.IsNullOrWhiteSpace(args.Result.LastPart))
		{
			return;
		}

		base.Process(args);
	}
}
```

Finally - create an include file that replaces the original processor with your new one: 
```xml
<?xml version="1.0"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<pipelines>
			<renderField>
				<processor type="Sitecore.Pipelines.RenderField.AddBeforeAndAfterValues, Sitecore.Kernel">
					<patch:attribute name="type">LM.Ignite.Core.Pipelines.RenderField.AddBeforeAndAfterValues, LM.Ignite.Core</patch:attribute>
				</processor>
			</renderField>
		</pipelines>
	</sitecore>
</configuration>
```

Now we simply check if the field contains anything or if we are in page edit mode. All bases covered! Now when our field is empty, the Before, After and EnclosingTag contents are not rendered to the output, Keeping out razor script markup nice and clean and not leaving empty html tags in our site!

Happy Coding!

-- Richard.