---
title: Sitecore.React - a new Sitecore module
date: 2016-08-02 07:58:02
tags:
- Sitecore
- ReactJS
- Marketplace
- Module
category: Sitecore.React
---
I recently wrote a series of posts on how we could use [React]() to create Sitecore components. React looks to be a great fit for Sitecore becuase just like Sitecore it is based around components that you can create and compose to make complex UIs. As an added benefit, using React to write the front end instead of Razor means that our front end teams do not have to know or even care about Sitecore. We can have a FED team that purely concentrates on building atomic components and as back end developers we can focus on developing the Sitecore part. This workflow can reduce the number of errors introduced when transfering the front end code into back end razor scripts.

Also - its pretty cool to be able to use a front end technology like React in a Sitecore environment.

## Dynamic Placeholders
As part of the roadmap I wanted to explore Dynamic Placeholders. As it turned out Jakob Christensen has already got a pretty neat solution for this with the [JsxRendering](https://github.com/JakobChristensen/Sitecore.Pathfinder/blob/master/src/Features/Sitecore.Pathfinder.React/Jsx/JsxRenderer.cs) in Pathfinder.

To add a placeholder to your `Jsx` component you simply need to use the `placeholders` object in the props for the component. Then give it the placeholder key you want to use. For example, if you wanted to create a placeholder called "content" you would use:

```javascript
{this.props.placeholder.content} 
```

So we have standard placeholders. This is the code that renders the placeholders. Notice we have a special prefix that allows dynamic placeholders:

```csharp
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
```

This means that to use a dynamic placeholder in `Jsx` we just need to prefix the placeholder key with `$Id.`. This then automatically appends the rendering unique Id to the placeholder key. 

```javascript
{this.props.placeholder.$Id.content}
```

For now, these dynamic placeholders are best used with the Experience Editor, as they are not predictable. In future versions there will be a switch to change the dynamic placeholder keys to predictable ones. This will make it easier to build presentation in the Content Editor.

## Sitecore Module
The examples so far have all been based around the Sitecore [Habitat](https://github.com/Sitecore/Habitat) project. That doesn't help for a lot of existing installations. So I have now submitted it as a Sitecore Module available in the Marketplace. You can view the module [here](https://marketplace.sitecore.net/Modules/S/SitecoreReact.aspx) once it has passed the review. For those of you that can't wait, the source for the module is on GitHub [https://github.com/GuitarRich/sitecore.react](https://github.com/GuitarRich/sitecore.react).

### Getting Started
The Sitecore module consists of 2 parts. The [Sitecore Package](https://github.com/GuitarRich/sitecore.react/raw/master/build/Sitecore%20Package/SitecoreReact-1.0.0.zip) contains all the templates required for the React renderings. The nuget package contains all the code to render the React components.

* First install the [Sitecore Package](https://github.com/GuitarRich/sitecore.react/raw/master/build/Sitecore%20Package/SitecoreReact-1.0.0.zip)
* Next install the NuGet package in your Visual Studio solution:
```
Install-Package Sitecore.React
```
* Now you can create your first React component. We will create a simple Title/Body component.

> SampleReactController.cs
```csharp
public SampleReactController : Controller 
{
    public ActionResult SampleReactRendering 
    {
        var data = new {
            Title = FieldRenderer(Sitecore.Context.Item, "Title"),
            Body = FieldRenderer(Sitecore.Context.Item, "Body")
        };

        return this.React("~/views/react/SampleReactRendering.jsx", data);
    }
}
```

Make sure that the `ActionResult` returns `this.React` - pass in the location to the `Jsx` file and the view model containing your data.


> SampleReactRendering.jsx
```javascript
var SampleReactRendering = React.createClass({
    render: function() {
        return (
            <div>
                <h1 dangerouslySetInnerHTML={{__html: this.props.data.Title}}></h1>
                <div dangerouslySetInnerHTML={{__html: this.props.data.Body}}></div>
            </div>
        );
    }
});
```

* Next we need to create the rendering item in Sitecore. Right click the folder where you want to create your rendering and select *Insert from Template*:
{% asset_img insertfromtemplate.png "Insert from template" %}

* Select a **React Controller Rendering**. Call it `SampleReactRendering`
{% asset_img reactcontrollerrendering.png "React Controller Rendering" %}

* Fill in the the fields in the rendering item:
    - **JSX File**: set this to the location of your jsx file
    - **Controller**: set to the controller class containing your rendering action
    - **Controller Action**: set to the `ActionResult` in your controller
{% asset_img samplereactrendering.png "SampleReactRendering" %}

* Finally, you need to make sure that the React JavaScript components and the React bundle added to the main layout:

```html
<script src="//fb.me/react-15.0.1.js"></script>
<script src="//fb.me/react-dom-15.0.1.js"></script>
@Scripts.Render(Sitecore.React.Configuration.Settings.ReactBundleName)
``` 

And that is it. Add your rendering to the presentation of an item and publish all the changes. The React component will be rendered Server-Side, the bundle will transpose the `Jsx` to `JavaScipt for any client-side rendering for state changes.

Please try it out and add comments here, or reach out to me on the [Sitecore Slack Community](http://sitecorechat.slack.com) - dm @guitarrich