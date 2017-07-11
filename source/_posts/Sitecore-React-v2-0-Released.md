---
title: Sitecore.React v2.0 - Released
tags:
  - Sitecore
  - ReactJS
category: Sitecore.React
date: 2017-06-26 21:36:30
---

Finally I have managed to get to launching version 2.0 of my [Sitecore.React](https://github.com/GuitarRich/sitecore.react) module. This post will highlight some of the new bits and also introduce the sample project built with Sitecore.React/. My hope is that this sample project will be to Sitecore.React what Habitat is to Sitecore - an example of how you can use Sitecore.React to build a project using ReactJS and Sitecore.

These are the updates I presented at the fabulous [SUCGON](http://www.sugcon.eu/) in Amsterdam this year. Thanks to all who stopped by to listen to the presentation. There was an issue with the recording of the demo, so I'm going to get a new recording done soon for any that care!

Over the next few weeks, I will add a series of posts and examples of how the project can be used and how this is one of the options to use ReactJS in a Sitecore environment.

The Nuget packages have been published. There are now 2 packages:

* [sitecore.react](https://www.nuget.org/packages/Sitecore.React/) : this is the core binary and can be referenced in any project
* [sitecore.react.web](https://www.nuget.org/packages/Sitecore.React.Web/) : this just contains the config files for use by the website project

# Sitecore.React v2.0 - What's New

## Placeholders

The big change in v2 is how Sitecore placeholders are handled. In v1 we handled placeholders as part of the prop data.

For example, a placeholder with a key of `main` was added like this:

```javascript
    {this.props.placeholder.main}
```

While this works, it means that for the React developer, the component stops there. For them to composite components together to build a functional React site, they would have to note down where the Sitecore placeholder should be added. This kinda kills one of the big benefits of using React, we don't want the backend developer to have to make any changes to the front end files.

So we now have a new `npm` package called `[sitecore.react.placeholders](https://www.npmjs.com/package/sitecore.react.placeholders)` - now adding a placeholder changes to:

```javascript
    <Placeholder placeholderKey='main' isDynamic={true} placeholder={this.props.placeholder}>
    </Placeholder>
```

As you can see, the placeholder is now a component and can have children. Now we can use this to build a component. Lets break down a typical Page Header component:

```javascript
import React from 'react';
import Placeholder from 'sitecore.react.placeholders';

var PageHeader = React.createClass({
    render() {
        return (
            <header class="page-header">
                <Placeholder placeholderKey={'page-header'} placeholder={this.props.placeholder}>
                    {this.props.children}
                </Placeholder>
            </header>
        );
    }
});

module.exports = PageHeader;
```

This is the jsx code for the component. First we include `React` and `Placeholder` - you can do this via the ES6 `import` statement or by using `require`.

Next we are defining the `PageHeader` class, in the render method we have added markup and a placeholder. Notice that nested in the placeholder is `{this.props.children}`. This tells the placeholder component to render any child elements of the current react component as child elements of the placeholder when running in Non-Sitecore mode.

When this component is rendered by Sitecore, the contents of the placeholder will be passed into the component by the Sitecore react module and the placeholder component will render that instead. In following posts I will expand how this happens.

**IMPORTANT** - you may have noticed the `placeholder={this.props.placeholder}`. This property must be passed into the placeholder component. It is part of how the component knows if it should render child elements or replace those with Sitecore content.

## JavaScript Bundling/Parsing

Another big change is how we are handling the JavaScript bundling for the project. Previously the jsx components were added into the bundle at runtime and we used ReactJS.Net to transpile the jsx to raw JavaScript. While this worked there were some disadvantages:

- **Performance:** Because the transpiling was done on the fly - there was a small performance impact. Also each page could have a different bundle identifier, meaning that you could not take advantage of browser caching for the JavaScript files easily.
- **Caching:** Again because the bundled js files did not exist prior to a request, we could not use a CDN to host the files. 
- **ES6/ReactJS.Net:** While the guys writing ReactJS.net have done an amazing job, not all features of ES6 are supported. The most annoying being modules. With pure ReactJS.Net transpiling you cannot use: `import React from 'react';` - you are forced to use `var React = require('react');` - its a small thing, but there are other small quirks like that. These are things that a pure ReactJS developer would expect to be in place, and so they shouldn't have to work around those things.

So to work around this `Sitecore.React` now supports pre-bundling of your jsx components. This is most commonly done using `[webpack](https://webpack.github.io/)` or another front end tool. The advantage of using `webpack` is that all of your components are created in the bundled JavaScript up front. Then we can use ReactJS.Net to render that pre-built component server side, saving us from having to transpile on the fly before each rendering.

It also means less differences from how the front end team are working, leading to fewer issues caused by differing processes.

There will be a section in the tutorials on getting started with webpack.

## Bugs and tweaks

There have been a few other tweaks and bugs fixed in the code too. Big thanks go out to [@Sean Holmesby
]() for many conversations about React and helping out in the early stages with this. Credits also to [Chris Nielson](https://github.com/altearius) and [Ryan Tuck](https://github.com/RyanTuckNZ) for contributing.

Also big thanks to [@Jason Wilkerson](https://twitter.com/LonghornTaco) for supplying a laptop at SUGCON so I could complete my presentation, as mine decided to die on the flight over!

Keep an eye out over the next few days and the tutorials will be coming and if you have any questions - please reach out. I'm on the [Sitecore Slack Chat](https://sitecorechat.slack.com) most days and go by the name **GuitarRich** - or hit me up on twitter [@rich_seal](https://twitter.com/rich_seal).

You can find the source to the module here: [https://github.com/GuitarRich/sitecore.react](https://github.com/GuitarRich/sitecore.react)
And the sample project here: [https://github.com/GuitarRich/sitecore.react.project](https://github.com/GuitarRich/sitecore.react.project) 