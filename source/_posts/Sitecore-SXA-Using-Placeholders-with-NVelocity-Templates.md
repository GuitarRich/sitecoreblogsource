---
title: 'Sitecore SXA: Using Placeholders with NVelocity Templates'
tags:
  - Sitecore
  - SxA
  - NVelocity
  - Rendering Variants
  - Placeholders
category: SxA
date: 2018-10-23 10:03:23
---

Recently I came across a situation where I needed to use an NVelocity template in a Rendering Variant. We were creating a custom slider and using the awesome [Slick Slider]() as it gives us a lot of nice options, and it has some good extras for accessibility that the OOTB Carousel component does not have at the time of writing. For the slide content, we still used a `Page Content` component and a custom Rendering Variant, but on the slide I wanted to make the `Slide Image` field a background image on the main `<div>` element. The second requirement was that the design called for 2 types of slide, one with just the background image, and one where we could add a child component into a placeholder. This could be used as a promo/call to action, but needed to be detached from the slide content. This would allow the client to personalize and test the call to action separately from the slide imagery.

Here is an example of the markup I needed to reproduce in the Rendering Variant:

```html
<div class="carousel-slide" style="background-image:url('/-/media/mybackgroundimage.jpg');">
    <div class="carousel-slide-content">
        @Html.Sitecore().Placeholder("carousel-slide-placeholder-key");
    </div>
</div>
```

The first problem is that currently OOTB you can't set an image field as a background image on a html tag. Fortunately there are some nice blog posts already out there on how to extend the NVelocity template:

* Michael West has post post on a [Custom Rendering Variant Token Tool for SXA](https://michaellwest.blogspot.com/2017/04/custom-rendering-variant-token-tool-for-sxa.html)
* Chaitanya Marwah hit prettly close with this post on [Design Sitecore SXA components using Variant Template](http://www.cmsitecore.com/2017/09/design-sitecore-sxa-components-using.html)

## Quick Reminder on how to Extend the Template

Just as a refersher on how to extend the template, first we need a new processor adding to the `getVelocityTemplateRenderers` pipeline. Implement `IGetTemplateRenderersPipelineProcessor` and add your code to the `Process` method:

```csharp
public class AddTemplateRenderers : IGetTemplateRenderersPipelineProcessor
{
    public void Process(GetTemplateRenderersPipelineArgs args)
    {
        Assert.ArgumentNotNull(args.Context, "args.Context");
        args.Context.Put("fieldTokens", new FieldTokens());
    }
}
```

And patch it in:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:set="http://www.sitecore.net/xmlconfig/set/">
    <sitecore>
        <pipelines>
            <getVelocityTemplateRenderers>
                <processor type="Foundation.SXA.Pipelines.Variants.GetVelocityTemplateRenderers.AddTemplateRenderers, Foundation.SXA" resolve="true" />
            </getVelocityTemplateRenderers>
        </pipelines>
    </sitecore>
</configuration>
```

Next, create the static method to get the url from an image field:

```csharp
public class FieldTokens
{
    public static string GetImageUrl(Item item, string fieldName)
    {
        var url = string.Empty;
        if (item != null && fieldName != string.Empty)
        {
            var field = (ImageField)item.Fields[fieldName];
            if (field != null)
            {
                url = ItemExtensions.ImageUrl(item, fieldName);
            }
        }

        return url;
    }
}
```

This now lets us create the Template definition with the following code:

```html
<div class="carousel-slide" style='background-image: url("$fieldTokens.GetImageUrl($item, "SlideImage")")'>
    <div class="carousel-slide-content">
        <!-- TODO: Add Placeholder Here -->
    </div>
</div>
```

## Creating the Rendering Variant

So the next task is to add the placeholder in the right place. This is where the problem started. There is no way to add a placeholder into a template out of the box. Adding a placeholder requires a valid view context, so trying to add one in the same way that we added the token for the image url would be overly complex.

[Michael West](https://twitter.com/MichaelWest101) came up with a great suggestion, in the template we create the start of the `<div>` markup. Like this:

```
<div class="carousel-slide" style='background-image: url("$fieldTokens.GetImageUrl($item, "SlideImage")")'>
```

I would set the `Tag` field to empty, so that there is no other markup added. Next we create a section that contains the placeholder items. Finally we can add a `Text` item in the variant definition with `</div>` in the text to close the earlier `div`.

Now all I need to do is to add the placeholder item in between those 2 items and I should be golden!

{% asset_img variantsetup.png "Rendering Variant Setup" %}

## Huston... We have a problem

Or so I thought... it turns out that even with the `Tag` field empty in a `VariantTemplate` item still adds a `div` surrounding the contents of the `Template` field. So the markup looked like this:

```html
<div>
    <div class="carousel-slide" style="background-image:url('/-/media/mybackgroundimage.jpg');">
</div>
    <div class="carousel-slide-content">
        @Html.Sitecore().Placeholder("carousel-slide-placeholder-key");
    </div>
</div>
```

Which is invalid and although most browsers helpfully try to close the div's, it didn't give me what I needed. After a bit of digging, the offending code can be found here in the `Sitecore.XA.Foundation.RenderingVariants.Pipelines.RenderVariantField.RenderTemplate` processor. Here is the `RenderField` method:

```csharp
    public override void RenderField(RenderVariantFieldArgs args)
    {
      Sitecore.XA.Foundation.RenderingVariants.Fields.VariantTemplate variantField = args.VariantField as Sitecore.XA.Foundation.RenderingVariants.Fields.VariantTemplate;
      if (variantField == null)
        return;
      HtmlGenericControl tag = new HtmlGenericControl(string.IsNullOrWhiteSpace(variantField.Tag) ? "div" : variantField.Tag);
      this.AddClass(tag, variantField.CssClass);
      this.AddWrapperDataAttributes((RenderingVariantFieldBase) variantField, args, tag);
      Dictionary<string, object> parameters = new Dictionary<string, object>()
      {
        {
          "item",
          (object) args.Item
        }
      };
      if (args.Parameters != null && args.Parameters.ContainsKey("geospatial"))
        parameters.Add("geospatial", args.Parameters["geospatial"]);
      tag.InnerHtml = ServiceLocator.ServiceProvider.GetService<ITemplateRenderer>().ExecuteTemplate(args.Item.Name, variantField.Template, parameters);
      args.ResultControl = (Control) tag;
      args.Result = this.RenderControl(args.ResultControl);
    }
```

You can see here on line 6, that if the `varitantField.Tag` property is null or white space, then a `div` tag will be defaulted too! So now I knew where the problem was, I can set about fixing it! 

First we need to override the `RenderTemplate` processor in the `RenderVariantField` pipeline. So create a new processor class and inherit from the existing one. Then override the `RenderField` method:

```c++
    public class RenderTemplate : Sitecore.XA.Foundation.RenderingVariants.Pipelines.RenderVariantField.RenderTemplate
    {
        public override void RenderField(RenderVariantFieldArgs args)
        {
            var variantField = args.VariantField as VariantTemplate;
            if (variantField == null)
            {
                return;
            }

            if (!string.IsNullOrWhiteSpace(variantField.Tag))
            {
                base.RenderField(args);
            }
            else
            {
                var templateRenderer = ServiceLocator.ServiceProvider.GetService<ITemplateRenderer>();
                var parameters = new Dictionary<string, object>() { { "item", args.Item } };
                if (args.Parameters != null && args.Parameters.ContainsKey("geospatial"))
                {
                    parameters.Add("geospatial", args.Parameters["geospatial"]);
                }

                // There is no surrounding html tag, so just render the resulting string from the template.
                args.Result = templateRenderer.ExecuteTemplate(args.Item.Name, variantField.Template, parameters);
            }
        }
    }
```

Now on line 12, we check to see if the `variantField.Tag` has been filled in or not. If it has, we pass this through to the base method and let it do its thing. If not, we will take over. We have to get the `templateRenderer` and build the `parameters` object to be able to render the field. But now we are not creating any surrounding tag, we are just executing the template and adding the resuling html to the `args.Result` property.

Now our variant renders exactly as I wanted it to, with a `div` containing a background image, the placeholder in the middle and then the closing `</div>` tag.

I'm not sure why the default behaviour is to force a `div` tag around template variants if the `Tag` field is left empty. But hopefully this option will help others who need to be a bit more creative with the rendering variant definitions!

Thanks to Michael West for helping out with the original idea!


--Richard