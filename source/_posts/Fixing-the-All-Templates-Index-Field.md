title: "Fixing the All Templates Index Field"
date: 2014-09-07 09:29:16
tags:
- Sitecore
- SOLR
category: Sitecore
---
One of the really nice features in [Fortis](http://fortis.ws/) is the ability to be able to search based on template type. Because we create an interface for every template in Sitecore, we can use the full inheritance tree to bring out the data we need.

A good example of this would be when building navigation. At Lightmaker we have a standard Navigation template. This template gives us all the fields we need to build navigational elements. Our main Page Type template, inherits from the Navigation Template. We can use this to get a list of items in Sitecore.

#### Fortis Method

``` csharp
public IEnumerable<INavigation> GetNavigation(IItemWrapper rootNode)
{
	using (var searchContext = ContentSearchManager.GetIndex("sitecore_web_index").CreateSearchContext())
	{
		return ItemFactory.Search<INavigation>(searchContext)
		       .Where(item => item.LongID.Contains(rootNode.ItemShortID)).ToList();
	}
}
```

#### Standard Sitecore Method

``` csharp
public IEnumerable<INavigation> GetNavigation(IItemWrapper rootNode)
{
	using (var searchContext = ContentSearchManager.GetIndex("sitecore_web_index").CreateSearchContext())
	{
		return
			searchContext.GetQueryable<SearchResultItem>()
				.Where(item => item.Templates.Contains("{AB86861A-6030-46C5-B394-E8F99E8B87DB}"))
				.ToList();
	}
}
```

## The Problem

This works fine out of the box. But there is a problem when the template you want to filter is further down in the inheritance tree. The standard GetAllTemplates method used for the AllTemplates computed field, only uses the item.TemplateID and item.Template.BaseTemplates. Consider the scenario where you have 3 templates, `Template A`, `Template B` and `Template C`. `Template B` inherits from `Template A` and `Template C` inherits from `Template B`. If we want do a search for all items that have `Template A` in the inheritance tree, only items that directly use `Template A` or `Template B` will be returned. The expected result is that items that use ANY of the 3 templates should be returned.

## The Solution

The solution is a custom AllTemplates computed field. This just iterates through all the base templates and builds the field.

``` csharp
using System;
using System.Collections.Generic;

using Sitecore.ContentSearch;
using Sitecore.ContentSearch.ComputedFields;
using Sitecore.ContentSearch.Utilities;
using Sitecore.Data.Items;
using Sitecore.Diagnostics;

namespace Lightmaker.Search.ComputedFields
{
	public class AllTemplates : IComputedIndexField
	{
		private Stack<TemplateItem> stack;
		private List<string> allTemplates; 

		public object ComputeFieldValue(IIndexable indexable)
		{
			var item = (Item)(indexable as SitecoreIndexableItem);

			this.allTemplates = new List<string> { IdHelper.NormalizeGuid(item.TemplateID) };
			this.stack = new Stack<TemplateItem>();
			foreach (var template in item.Template.BaseTemplates)
			{
				this.stack.Push(template);
			}

			while (this.stack.Count > 0)
			{
				var templatesToProcess = this.ProcessTemplate(this.stack.Pop());
				foreach (var t in templatesToProcess)
				{
					this.stack.Push(t);
				}
			}

			return this.allTemplates;
		}

		private IEnumerable<TemplateItem> ProcessTemplate(TemplateItem templateItem)
		{
			if (templateItem == null)
			{
				return new TemplateItem[0];
			}

			this.allTemplates.Add(IdHelper.NormalizeGuid(templateItem.ID));
			return templateItem.BaseTemplates;
		} 

		public string FieldName { get; set; }
		public string ReturnType { get; set; }
	}
}
```

Notice that we are not using recursion here to build the list of templates. We could, but as this code gets called every time an item is indexed, recursion would be a bottle neck here. Instead we are using a stack object and iterating through the list. We could also make this code more robust and check for circular references, as that would cause an out of memory error eventually.

The final part of the puzzle is to setup the configuration, this would be the include file to use the new AllTemplates computed field:

``` xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<contentSearch>
			<indexConfigurations>
				<defaultSolrIndexConfiguration>
					<fields hint="raw:AddComputedIndexField">
						<field fieldName="_templates" returnType="string">Lightmaker.Search.ComputedFields.AllTemplates, Lightmaker.Search</field>
					</fields>
				</defaultSolrIndexConfiguration>
			</indexConfigurations>
		</contentSearch>
	</sitecore>
</configuration>
```

Coming up, more articles on how we use Fortis to streamline our Sitecore development and how we implemented a real time breaking news module using SignalR.

– Richard Seal
