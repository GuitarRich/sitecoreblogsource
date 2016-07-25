title: "Securing the Assets data folder"
date: 2015-11-30 17:14:10
tags:
category: Sitecore
---

There was an interesting conversation on the Sitecore Slack Community about where the best place to store *single-use* data items for components. The consensus was that a folder that sits under the page item is generally the best place to put it, it makes it easy to find for the content editors and simple to package up a single page and deploy elsewhere. This is the method that we have been using at [Lightmaker](http://www.lightmaker.com) for a while now and I know a number of other agencies use a similar practice.

But as [Jason St-Cry](https://twitter.com/AgileStCyr) pointed out, it does have the side effect that the asset folder and data items become urls that can be resolved by Sitecore. Most of the time these items do not have any presentation associated with them, but that just means that they will throw a `LayoutNotFoundException` where really any urls that point to data items, should really throw a 404.

###Folder Structure
Lets have a look at the folder structure in place here:

{% asset_img folderstructure.png "Folder Structure" %}

You can see here, under each `Page Item` there is an `Assets` folder. In here we can create all the content required for that page, where the content is only for use on that page. (See here for tips on styling the content tree - [Style Items for the Sitecore Content Editor tree - Jason Bert](http://www.jasonbert.com/2015/01/23/tip-style-items-sitecore-content-editor-tree/))

Normally we will have a shared content folder somewhere in the site also.

To help the editors, we can set these paths in the `Datasource Location` field on the `Rendering Item`, notice the query for the path, this helps in a multi-site solution.

{% asset_img datasourcelocation.png "Datasource Location and Template settings" %}

We can also set the `Datasource Template` field to enable the content editor to create content easily from the page editor.

So how do we filter these content items out of the request so that any urls that reference them become 404 errors? 

###Time for a Custom Pipeline Processor!
We can do this pretty simply with a custom processor right after the `ItemResolver` in the `httpRequestBegin` pipeline. First we need to be able to identify the items that are *not* page items. To do that we have a base template that all our templates inherit from, we add a field to this to identify if the item should resolve a url or not. This field is set to true in the Standard Values of the base template.

{% asset_img basetemplate.png "Base Template item resolution field" %}

Now for each template that is pure data, we can simply untick the `Resolve to Url` field in the standard values of the template. 

###Procesor

Here is the code for the processor:

```csharp
namespace MyProject 
{
	using Sitecore;
	using Sitecore.Data.Fields;
	using Sitecore.Diagnostics;
	using Sitecore.Pipelines.HttpRequest;
	
	public class ContentOnlyItemResolver : HttpRequestProcessor
	{
		public override void Process(HttpRequestArgs args)
		{
			Profiler.StartOperation("Resolve any Content only items.");
	
			Assert.ArgumentNotNull(args, "args");
			if (Context.Item == null || Context.Database == null || args.Url.ItemPath.Length == 0)
			{
				// Only check this if we have a valid context item
				return;
			}
	
			// Check the "Resolve to Url" field - if the field is false or doesn't exist, we
			// can just carry on as normal.
			CheckboxField resolveToUrl = Context.Item.Fields["Resolve to Url"];
			if (resolveToUrl == null || resolveToUrl.Checked)
			{
				return;
			}
	
			// TODO: Implement a 404 here, or alternatively set the Context.Item to 
			// null and let the Sitecore or other 404 processor take care of the redirect
			Context.Item = null;
	
			Profiler.EndOperation();
		}
	}
}
```

and then we can simply patch it in with an include file. Make sure it is patched **after** the `ItemResolver` but before any 404 processors:

```xml
<?xml version="1.0"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<pipelines>
			<httpRequestBegin>
				<processor patch:after="processor[@type='Sitecore.Pipelines.HttpRequest.ItemResolver, Sitecore.Kernel']"
						   type="MyProject.ContentOnlyItemResolver, LM.Ignite.Core">
				</processor>
			</httpRequestBegin>
		</pipelines>
	</sitecore>
</configuration>
```

Now when we visit a standard page item, we get the content as we should, but try to visit any of the *non*-page items, or anything that has the `Resolve to Url` field un-checked, a 404 will be returned.

###Why not just check the template/base templates
That was an option that we thought of, but doing it with a field means we can override on an item by item basis if we decide for some reason that one or more of the content items need to have presentation and resolve.

Hope this helps someone!

--Richard Seal