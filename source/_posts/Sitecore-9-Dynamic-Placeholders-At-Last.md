---
title: 'Sitecore 9: Dynamic Placeholders At Last!'
tags:
  - Sitecore 9
  - Dynamic Placeholders
  - Sitecore
category: Sitecore 9
date: 2017-10-17 08:00:00
---

{% asset_img soexcited.gif "I'm so excited" %}

Hell yeah - finally dynamic placeholders come out of the box with Sitecore 9. This has probably been one of the most asked for features in Sitecore for a long time. Fortunately there have been a few community modules that have helped in the past. One of the most popular was the [Fortis Dynamic Placeholders](https://github.com/Fortis-Collection/dynamic-placeholders) which used the rendering item's unique Id to generate the placeholder key.

## So what does Dynamic Placeholders give us now?

At the basic level, you can use dynamic placeholders in the same way as the Fortis ones. Just by adding one to your rendering:

```html
@Html.Sitecore().DynamicPlaceholder("placeholderKey")
```

But that is just the start of it. Let's have a deeper dive into some of the more advanced features of the new Dynamic Placeholders:

## Multiple Placeholders

If we look at the overloads for dynamic placeholders we see that there are parameters for `count` and `seed`.

```csharp
@Html.Sitecore().DynamicPlaceholder(string placeholderName, int count = 1, int maxCount = 0, int seed = 0);
```

Looking at the documentation `count` specifies how many placeholders Sitecore will render for that key, `seed` specifies the starting number that the suffix will use and `maxCount` limits the number of placeholders that Sitecore will create. Sitecore will use the format: `{placeholder key}-{rendering unique suffix}-{unique suffix within rendering}`. For example:

```text
content-{7a943e27-b649-400c-986d-33d07f0f50ca}-1
```

These extra parameters make the new Dynamic Placeholders very flexible. They no longer just deal with a component containing a placeholer being able to be added to the page multiple times, but also creating multiple placeholders in the same component. We will dig deeper in to why you might want to do that later.

## Modifying the Chrome

Another look at the overloads and we can see that there are some methods that allow us to control the markup that wraps each placeholder created by this one call.

First you can pass a `TagBuilder`, this is handy, but what if you need to know something about the context of this placeholder? For example the index of this placeholer. Well you can also supply the call a function that returns a `TagBuilder` object. That function also has a `DynamicPlaceholderRenderContext` object passed in. This gives you detail about that context of each placeholder as it is added.

Let's say that you wanted to build an accordion component, but instead of limiting the content of each accordion, we want to set the accordion content using a placeholder so that we can dynamically build that content. Historically, you would build the markup for the accordion and in the center add your dynamic placeholder implementation.

```html
@foreach(var accordionItem in Model.AccordionItems) {
    <div class="collapse accordion-@(accordionItem.Index)">
        Html.Sitecore().Placeholder("accordion-" + accordionItem.Index)
    </div>
}
```

We could make this simpler by using the `TagBuilder` like this:

```csharp
@Html.Sitecore().DynamicPlaceholder("accordion", context => {
    var tag = new TagBuilder("li");
    tag.AddCssClass("collapse")
    tag.AddCssClass("accordion-" + context.Index)
    return tag;
}, 3);
```

This will then generate 3 `li` tags with the class set to `collapse accordion-1` etc... 

But the `TagBuilder` can make the markup difficult to read, maybe we want to use more readable markup. We can also do that by using the overload with the `outputModifier` - this allows us to build a `HtmlString` object and return it.

Finally you can also pass in a `DynamicPlaceholderDefinition` - in both of these you can specify the mark up you want to use to wrap the contents of the placeholder. Here is the above example using that:

```csharp
@Html.Sitecore().DynamicPlaceholder(new DynamicPlaceholderDefinition("accordion")
{
    Count = 3,
    MaxCount = 10,
    Seed = 1,
    OutputModifier = (input, context) => new HtmlString("<li class=\"collapse accordion-" + context.Index + "\">" + input + "</li>")
})
```

## Bringing this to life

So why is all of this so cool? Well, lets imagine you want to create an accordion or maybe a tabbed component. The content editor will need to be able to specify the number of accordion elememts or tabs and then using the Experience Editor, build out the content.

Using rendering parameters and dynamic placeholders we can achieve that. Dynamic Placeholders will automatically look for properties on the rendering parameters that follow this pattern: `ph_placeholderKey_paramName`. This means that to override things like `count`, `seed` etc.. you just have to add an additional property to the component properties like this:

{% asset_img renderingparams.png "Additional Properties" %}

Now 10 placeholders will be created. Note that adding this additional property _will_ override the setting you specified in the razor view, with the exception of the `MaxCount` property. So if you want your content editors to be able to add/remove placeholders via component properties, it is important to remember to set limits so that things can't get too crazy.

## Important: Things to remember

There are some things to remember when using dynamic placeholders. First, the placeholder only knows about its own call. It does not know about any other dynamic placeholders that might be on the page. Take the following code:

```csharp
<div id="myComponent">
    <h2>Section 1</h2>
    @Html.Sitecore().DynamicPlaceholder("column", 2)

    <h2>Section 2</h2>
    @Html.Sitecore().DynamicPlaceholder("column", 2)
</div>
```

This would be the equivalent of:

```csharp
<div id="myComponent">
    <h2>Section 1</h2>
    @Html.Sitecore().Placeholder("column_{...}_0")
    @Html.Sitecore().Placeholder("column_{...}_1")

    <h2>Section 2</h2>
    @Html.Sitecore().Placeholder("column_{...}_0")
    @Html.Sitecore().Placeholder("column_{...}_1")
</div>
```

The second call to the dynmic placeholders does not know anything about the first. To do this you would need to set the `seed` property:

```csharp
<div id="myComponent">
    <h2>Section 1</h2>
    @Html.Sitecore().DynamicPlaceholder("column", count: 2, seed: 100)

    <h2>Section 2</h2>
    @Html.Sitecore().DynamicPlaceholder("column", count: 2, seed: 200)
</div>
```

Now your placeholders will not conflict.

## Updating from existing solutions

One big question is going to be - How can I update my existing solution to use the Sitecore version? Well stay tuned, that will be the subject of another post.

-- Richard