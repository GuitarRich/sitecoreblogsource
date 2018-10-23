---
title: SXA - Debunking the Myths
tags:
- Sitecore
- SXA
category: SxA
---
When chatting with other developers/managers/interested parties about SXA there are a few common misconceptions that I see put up as reasons why SXA shouldn't be used on a particular project. In this post I'll attempt to address some of the more common issues I have heard and hopefully debunk some of the myths that seem to have gained traction about SXA.

## So should I be using SXA in my project?

The TL/DR for this is, does the client have a license for it? Or, does the client have a subscription based Sitecore 9 license. If they do, then in my experience SXA is a winner almost every time (I say almost, because there are always edge cases). IMO SXA turns Sitecore into the product it always should have been, and while its not perfect and could be improved in some areas. It really does create a nice experience for the Marketer/Content Editor and also helps to streamline development.

... and no... I'm not getting paid to say that!

So lets start addressing some of the "blockers" that I commonly hear about why SXA should _not_ be used on a project!

## No 1: Front end hate it, There is no control over the markup

This is single handedly the biggest complaint I hear about SXA all the time. There is no control over the markup, Frontend have to fight the given markup and can't work with it. The markup SXA generates is bad.

{% asset_img wrong.gif "Fake News" %}

Ok, while this may seem like the case when you first look at SXA, _and_ is true for a small number of components, its absolute rubbish for the majority of components.

Why? Rendering Variants! Nearly all of the pre-built components in SXA make use of the Rendering Variant system. This means that aside from a small bit of scaffolding markup, the main section of the markup for that rendering is fully controllable... from within Sitecore. Its just a _different_ way of building the markup from the usual Sublime/VS Code editors that the front end team might be used to.

Lets take a look at one of these components, the simplest, a `Promo`. Here is the razor view for the `Promo` component

```html
@using Sitecore.XA.Foundation.MarkupDecorator.Extensions
@using Sitecore.XA.Foundation.RenderingVariants.Extensions
@using Sitecore.XA.Foundation.RenderingVariants.Fields
@using Sitecore.XA.Foundation.SitecoreExtensions.Extensions
@using Sitecore.XA.Foundation.Variants.Abstractions.Fields
@model Sitecore.XA.Foundation.Variants.Abstractions.Models.VariantsRenderingModel

@if (Model.DataSourceItem != null || Html.Sxa().IsEdit)
{
    <div @Html.Sxa().Component(Model.Rendering.RenderingCssClass ?? "promo", Model.Attributes)>
        <div class="component-content">
            @if(Model.DataSourceItem == null)
            {
                @Model.MessageIsEmpty
            }
            else
            {
                foreach (BaseVariantField variantField in Model.VariantFields)
                {
                    @Html.RenderingVariants().RenderVariant(variantField, Model.Item, Model.RenderingWebEditingParams)
                }
            }
        </div>
    </div>
}
```

Here you can see the basic scaffolding elements. There are 2 divs, the outer contains the classes and attributes that identify the component to SXA and to the Theme. This allows our front end team to namespace the CSS Classes to be specific to a particular component. This is important as it then means that the component can be used in multiple places in your site build. It is not limited to one single structure because of bad markup/css design.

The inner `div` just lets us know that the content of the component is comming next. TBH - we could _probably_ do without that extra `div`, but it doesn't do anyone any harm by being there. 

Once inside the inner `div`, you can see a loop that renders the markup out from the Rendering Variant fields created for the component in Sitecore. 

### What is the problem then?

There are a few misconceptions I have seen about these rendering variants and they are either caused by unfamiliarity with the product or a lack of communication between the back end and front end development teams. It also changes the approach to the creation of the front end work. Traditionally, this would happen in static files before the main backend work is done. Now, that is slightly reversed, although a lot can now happen in parallel - reducing the time to market of the project.

Once the front developer is aware that they can structure the HTML as they want to, they are a lot happier, it just takes a TEAM effort then to coordinate updates being made in Sitecore and then exported by Creative Exchange. 

Here is an example of how the markup is setup within Sitecore as a Rendering Variant:

#TODO: ADD RENDERING VARIANT SCREEN SHOT HERE

