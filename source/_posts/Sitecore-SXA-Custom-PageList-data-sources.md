---
title: 'Sitecore SXA: Custom PageList Datasources'
tags:
  - Sitecore
  - SxA
  - Datasource
category: SxA
date: 2018-03-16 11:54:13
---

Sitecore Experience Accelerator ([SXA](https://doc.sitecore.net/sitecore_experience_accelerator)) has so many cool features to help us as developers to create amazing Sitecore implemnentations.

{% asset_img amazing.gif "Sitecore SXA - Amazing!" %}

One component that I find myself using a lot, is the `Page List`. This component allows a marketer to display a list of pages from the site. This could be a list of products, blog pages, news articles etc... We can configure how the page list gets its list of pages by either setting a datasource or setting the `Source Type` field.

{% asset_img sourcetype.png "Set the Source Type on a Page List" %}

The Source Type is build from a list of `Query` items that can be found under `Site\Settings\Item Queries`. This item contains a `DataSource` field that allows a developer to define what items should appear. We could set a root item, or maybe use a Search Query in there. [Gert Gullentops](https://sitecore.stackexchange.com/users/237/gatogordo) has a nice article on using the [Page List component & Item Queries](https://ggullentops.blogspot.com/2017/04/sitecore-sxa-pagelist-item-query.html) that details how you can set those up and how powerful they can be.

{% asset_img itemqueries.png "Item Queries" %}

Unfortunately, for my needs, they did not work. I had 2 requirements:

1. I needed to show a list of sibling items that were of the same template as the context item, _BUT_ excluded the current context item.
1. I needed to show a list of items that were related to the current context item by matching any of the tags selected.

Although the default source types contains an `Item Query` for `Siblings` - when set to this, it does not exclude the current context item. Thank you to [Alan Płócieniak](https://sitecore.stackexchange.com/users/16/alan-p%C5%82%C3%B3cieniak) on the Sitecore Slack channel for introducing me to an undocumented gem: 

## `code:` datasources.

A code datasource is pretty self explainitory - it uses some custom code to resolve the datasource item. To use it, you can set the DataSource field to a class that implements `Sitecore.Buckets.FieldTypes.IDataSource`. This interface has a single method to implement:

```csharp
public interface IDataSource
{
    Item[] ListQuery(Item item);
}
```

This method allows you to get an array of items that you want to use for your datasource. The `Item` passed in, will depend on where you have set the datasource. If you are using this on an `Item Query` or just setting the datasource of a component, it will be the `Context.Item`. This will be invoked in the `resolveRenderingDatasource` pipeline processor `Sitecore.XA.Foundation.LocalDatasources.Pipelines.ResolveRenderingDatasource.CodeDatasource`. 

So lets look at some code that will give us the results we needed for our first requirement. All siblings of the same template, excluding the current item:

```csharp
using System.Linq;
using Sitecore.Data.Items;
using Sitecore.XA.Foundation.SitecoreExtensions.Extensions;

namespace MyProject.Feature.Navigation.CodeDatasources
{
    public class SiblingsExcludingContextItem : Sitecore.Buckets.FieldTypes.IDataSource
    {
        public Item[] ListQuery(Item item) => 
            item.GetSiblings().Where(sibling => sibling.TemplateID == item.TemplateID).ToArray();
    }
}
```

Nice and simple. `.GetSiblings()` is a nice little extension method in the `Sitecore.XA.Foundation.SitecoreExtensions.Extensions` namespace, that calls:

```csharp
item.Parent.GetChildren().Where(sibling => sibling.ID != item.ID).ToList();
```

So that already excludes our context item, as that is what we are using to call the `GetSiblings()` method. When we just add a filter on to make sure the template ID matches and return the array of items.

Now we need to use it in our page list. For this we will use an `Query` item and set the source type. This will make it simple for marketers to add this type of filter on any page. So lets create a new `Query` item in the `site\Settings\Item Queries` folder. And set the datasource field to our code above. The value would be `code:MyProject.Feature.Navigation.CodeDatasources.SiblingsExcludingContextItem, MyProject.Feature.Navigation`

{% asset_img codedatasource.png "Creating the Code Datasource Item Query" %}

Now on my Page List component, I can just set the source type to `Sublings Excluding the Current Item` and it works like a dream!

So what about building a Page List code datasource that will show us related items by tag? Let's keep this simple by assuming that we only want to get related siblings of the current item:

```csharp
    public class SiblingsWithMatchingTag : Sitecore.Buckets.FieldTypes.IDataSource
    {
        public Item[] ListQuery(Item item)
        {
            var sourceItemTagIds = GetTagIds(item);
            return item.GetSiblings().Where(
                sibling => 
                    sibling.TemplateID == item.TemplateID 
                    && GetTagIds(sibling).Any(t => sourceItemTagIds.Contains(t)))
                        .ToArray();
        }

        private static IEnumerable<ID> GetTagIds(Item sourceItem)
        {
            if (sourceItem == null)
            {
                throw new ArgumentNullException(nameof(sourceItem));
            }

            return ((MultilistField)sourceItem.Fields[Foundation.SXA.Templates.Taggable.Fields.SxaTags])?.TargetIDs 
                ?? Enumerable.Empty<ID>();
        }
    }
```

Again, nice and simple, just create another `Query` item and set the `Source Type` on the page list. If you wanted to get a little more sophisticated, you could modify the above code to use the search API to pull out related items from the search index, this would perform well and also not limit you to only siblings.

### Posibilities

The posibilities are huge here. There are normally a few times when implementing a Sitecore site where you need to list pages, and its rare that its always as simple as just child pages, or items of the same template. We normally will want to manipulate the data in some way. These `code:` datasources provide a nice and simple way of doing that, while still using the standard SXA components.

Also - this doesn't have to be limited to only using the `Page List` component. We can use `code:datasources` with other components too. If you come up with some cool ways of using the `code:datasource`, link to it in the comments! Enjoy!

### Update: Confused about Datasources? You will be!!

So it seems there has been some confusion about whether this is a "new" feature of SXA or whether it has been around for a while (Since Sitecore 7). 

The answer is.... **BOTH**

As [Kamruz](https://twitter.com/jammykam) accurately [points out](https://twitter.com/jammykam/status/974824110822907904), the `code:Datasource` interface (`Sitecore.Buckets.FieldTypes.IDataSource`) has been around since Sitecore 7, there are some good posts about how it was originally used by [John West](https://community.sitecore.net/technical_blogs/b/sitecorejohn_blog/posts/sitecore-7-custom-classes-as-data-template-field-sources) and [Brent Svac](https://horizontalintegration.blog/2015/04/02/sitecore-source-field-code-query-that-implements-idatasource/). 

Kamruz also mentions an option using Sitecore PowerShell Extensions to do the same thing: [PowerShell Scripted Datasources](https://blog.najmanowicz.com/2013/04/17/powershell-scripted-datasources-in-sitecore-part-1/).

So what is the big deal? The confusion here lies in the way that the term `Datasource` can mean multiple things in Sitecore.

#### Field Datasource

A field datasource, is the source property of a field defined on a data template. These can be used for many things, depending on the field type. From selecting the Rich Text Editor profile, to setting the root element of a `TreeList`:

{% asset_img fielddatasource.png "Field Datasource" %}

#### Component Datasource

A component datasource, is the source that you set on a rendering to define where the data is pulled from for that rendering.

{% asset_img componentdatasource.png "Component Datasource %}

The blog posts above are talking about **Field Datasources**, the new bit here provided by the SxA pipeline processor, is that we can now use the same method for a **Component Datasource**. Yay - we all win. Both options are very useful in implementing Sitecore.


--Richard