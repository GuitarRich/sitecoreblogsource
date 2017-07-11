---
title: Sitecore.React - Getting Started - 1. Webpack
date: 2017-07-04 09:35:00
tags:
- Sitecore.React
- Sitecore
- ReactJS
- node.js
category: Sitecore.React
---
Welcome to the Sitecore.React getting started guide. In this guide we will walk through getting webpack setup and the structure of our site. Over the next few tutorials we will build the front end site and then bring it all over to Sitecore!

# Getting Started with Webpack

## Prerequisites

For this tutorial you need to have the following installed on your machine:
- [Node.js](https://nodejs.org/en/)

I am also going to assume a level of knowledge on [webpack](https://webpack.js.org/) and [reactjs](https://facebook.github.io/react/). If you want to find out more on webpack - take a look at the [getting started guide here](https://webpack.js.org/guides/getting-started/). You can get some good info on [reactjs](https://facebook.github.io/react/) [here](https://facebook.github.io/react/docs/hello-world.html).

There will be features of [ES6](http://es6-features.org) used too. You can see an [overview](https://github.com/lukehoban/es6features) of ES6 [here](https://github.com/lukehoban/es6features). You will also hear the word [transpile](https://scotch.io/tutorials/javascript-transpilers-what-they-are-why-we-need-them) often - have a read of the article linked if you are unsure [what a transpiler does](https://scotch.io/tutorials/javascript-transpilers-what-they-are-why-we-need-them).

This tutorial will be based around how we can create the [sitecore.react.project](https://github.com/GuitarRich/sitecore.react.project) sample project. For those of you that have cloned it already, the repo is not fully complete yet. I will do my best to get the serialized items and the readme updated asap.

## Basic Setup

There are a few main components of a Sitecore.React web application. For the purposes of this tutorial we will assume that you are building a full web application with React. If you are just using it for a few components and mixing that up with standard MVC Controller and View renderings, the basics are still the same, just without the main front end application part.

For the React components and front end build we will use webpack to transpile, bundle and minify the javascript. All the react components will be bundled into a single javascript file. To help with the differences between the front end react site and the Sitecore site we will build 3 versions of the main JavaScript file:

- **fed.js**: This is the main source for the react front end application. It will be where we setup react, the FED router and our application that runs outside of Sitecore
- **client.js**: This will be the main source for all JavaScript that is used in the Sitecore implementation. It will be referenced in the main layout and downloaded to the users browser. It will contain all the react components along with all the js for the application to run
- **server.js**: This is the main source for all server side JavaScript. It will contain all the react components that ReactJS.Net needs to know about. Alot will be identical to the **client.js** file, but this file will exclude any DOM manipulation done outside of react. e.g. event handlers, jQuery DOM manupulation etc... That is only ever in the **client.js** as our server side rendering does not need to know about it.

The front end react site will also conform to some standards to make the transition of the react files to Sitecore seamless. 

- First, we will setup a static data store that the front end team can use to populate the react site with content before it is delivered by Sitecore. The structure of each data model should be defined and agreed up by both front and back end teams.
- The front end team will use a Sitecore placeholders module and the locations of the placeholders should also be agreed upon with input from Sitecore developers.

It should become clear when going through the tutorials where each part fits in, so lets get on with configuring webpack and creating the obligatory **Hello World** application!

## Setup Webpack

First lets create a directory, initialize npm and install webpack locally. For this tutorial we will assume you start in `C:\projects`:

```bash
mkdir sitecore.react.project && cs sitecore.react.project
npm init -t
npm install --save-dev webpack
```

Now we want to setup our project structure. The finished project will have the following folder structure:

{% asset_img 01-FolderSetup.png 'Project Folder Setup' %}

The `src/App` folder will contain all the React components. `src/Feature`, `src/Foundation` and `src/Project` contain all the helix modules for the project.

To setup webpack we use the `webpack.config.js` file in the root of the project. Create this file and we will configure it like this:

```javascript
var debug = process.env.NODE_ENV !== "production";
var webpack = require('webpack');
var path = require('path');

module.exports = {
  context: path.join(__dirname, "src/app"),
  devtool: debug ? "inline-sourcemap" : null,
  entry: {
    fed: "./fed.js",
    client: "./client.js",
    server: "./server.js"
  },
  module: {
    loaders: [
      {
        test: /\.json$/,
        loader: 'json'
      },
      {
        test: /\.jsx?$/,
        exclude: /(node_modules|bower_components)/,
        loader: 'babel-loader',
        query: {
          presets: ['react', 'es2015', 'stage-0'],
          plugins: ['react-html-attrs', 'transform-decorators-legacy', 'transform-class-properties'],
        }
      }
    ]
  },
  resolve: {
    extensions: ['.js', '.jsx']
  },
  output: {
    path: __dirname + "/src/app",
    filename: "[name].min.js"
  },
  plugins: debug ? [] : [
    new webpack.optimize.DedupePlugin(),
    new webpack.optimize.OccurenceOrderPlugin()
  ],
};
```

Lets break that down, most of it is a pretty standard webpack config. The bits to notice are:

```javascript
  entry: {
    fed: "./fed.js",
    client: "./client.js",
    server: "./server.js"
  },
```

Here we are defining 3 entry points for webpack. This tells webpack that we are using those 3 files as our source for bundling. We have 3 seperate files for all `Sitecore.React` projects. These are the same files mentioned earlier: `fed.js`, `client.js` and `server.js`.

We will create these files in the next step.

```javascript
  output: {
    path: __dirname + "/src/app",
    filename: "[name].min.js"
  },
```

This section just defines the output files generated by webpack. Using `[name]` means that it will generate 3 files that match the entry files names with `.min.js` appended.

## Setting Up the React Application

It's not the purpose of this tutorial to teach how to write a reactjs application. There are plenty of resources already out there for that. What we will do is use the [sitecore.react.project](https://github.com/GuitarRich/sitecore.react.project)

Now we can start with out react application. The application is made up of the main entry point - this sets up the react router and some routes.

Make sure that your `package.json` file contains the following devDependencies:

```javascript
  "devDependencies": {
    "babel-core": "^6.17.0",
    "babel-loader": "^6.2.0",
    "babel-plugin-add-module-exports": "^0.1.2",
    "babel-plugin-react-html-attrs": "^2.0.0",
    "babel-plugin-transform-class-properties": "^6.3.13",
    "babel-plugin-transform-decorators-legacy": "^1.3.4",
    "babel-preset-es2015": "^6.3.13",
    "babel-preset-react": "^6.3.13",
    "babel-preset-stage-0": "^6.3.13",
    "bootstrap": "^3.3.7",
    "expose-loader": "^0.7.3",
    "jquery": "^3.2.1",
    "jquery-ui": "^1.12.1",
    "react": "^15.6.1",
    "react-bootstrap": "^0.31.0",
    "react-dom": "^15.6.1",
    "react-router": "^4.1.1",
    "react-router-dom": "^4.1.1",
    "webpack": "^3.0.0",
    "webpack-dev-server": "^2.5.0"
  }
```

Then run `npm install` in the root of your project.

Next lets create our `index.html` file. This will contain the main dom elements that our react application will be rendered too. You can add your standard headers, css files etc... In the body make sure we have an app div element and include bundled fed javascript file:

```html
<body>
    <div id="app"></div>
    <script src="/fed.min.js"></script>
</body>
```

Now create the file `fed.js` in the `src/App` folder. We will import jQuery globally and add in React and the React Router:

```javascript
global.jQuery = require('jquery');
require('bootstrap');
require('jquery-ui');

import React from "react";
import ReactDOM from "react-dom";
import { Router, Route, IndexRoute, hashHistory } from "react-router";
```

Now we can initialize the react application and render it to the div we created earlier:

```javascript
const app = document.getElementById('app');
ReactDOM.render(
    <h1>Hello World</h1>,
app);
```

As a final step - lets enable the webpack dev server. Open your `package.json` file and add the following entry to the `scripts` section:

```javascript
    "dev": "webpack-dev-server --content-base src/App --inline --hot"
```

Finally we will run `npm run dev` in the console. This will run the webpack dev server on port 8080. Now in your browser go to http://localhost:8080/ - if everything has worked you should see the hello world text being rendered by react:

{% asset_img helloworld.png %}

In the next tutorial we will start creating React components and learn how to setup placeholders.

-------

This is the first article in a series of tutorials on Sitecore.React:

1. {% post_link Sitecore-React-Getting-Started %}
1. {% post_link Sitecore-React-Getting-Started-Creating-Components %}
1. {% post_link Sitecore-React-Getting-Started-3-Data %}

If you have any questions - feel free to hit me up on [Sitecore Slack](https://sitecorechat.slack.com) - my user is @GuitarRich!