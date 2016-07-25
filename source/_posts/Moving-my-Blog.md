title: "Moving My Blog - Adventures in Hexo"
date: 2015-05-20 22:05:00
tags:
- Hexo
- Blog
- Visual Studio Code
category:
- General
---

I've had a pretty busy start to the year, moving house, visiting family back in the UK and at Lightmaker we have been launching some of the biggest implementations in the companies history! Combined with trying to get an [Orlando Sitecore User Group](http://www.meetup.com/CodeRebase-Orlando/events/220148692/) up and running, its been a whirlwind, but fun at the same time!

Unfortunately my blogging and efforts on Stack Overflow have been suffering, so in an attempt to force myself into writing again, and partly inspired by [Kams post on using Hexo](http://kamsar.net/index.php/2015/04/Blogging-with-Hexo-a-Node-js-detour/ "Kams post on using Hexo") - I'm giving it a try! So welcome to my Blog's new home!

You can read Kams post for some of the benefits of blogging with a Static Site Generator. Here are some of my observations:

###Setup
So simple!

1. Install [node.js](https://nodejs.org/)
2. Follow the commands on the [Hexo](http://hexo.io) home page:

``` bash
$ npm install hexo-cli -g
$ hexo init
$ npm install
$ hexo server
```
Point your browser at: `http://127.0.0.1:4000` and you will see the default Hello World post thats installed by the Hexo installer.

Less that 5 minutes to install, setup and see the first post!

###Git Integration
One really nice thing about Hexo is that you can generate the files and deploy with a single command. Just install one of the [deployment plugins](http://hexo.io/docs/deployment.html "Hexo Deployment Plugins").

I'm using Github Pages to host my blog, so I used [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git). In my _config.yml I just ad to add the deployment settings:

``` js
deploy:
	type: git
	repo: <repository url>

```

Now when a post is ready I just run:

``` bash
$ hexo generate --deploy
```
And all the files are generated and deployed automatically! You can see the source files for the repo right in your Github repo: [https://github.com/GuitarRich/guitarrich.github.io](https://github.com/GuitarRich/guitarrich.github.io).

###Create a New Post
Creating a new post is as simple as:

``` bash
$ hexo new post "Title of Post"
```

This creates a new file in _/source/_posts/Title-of-Post.md_ file and I have it configured to create a matching folder to add any images or other assets for that post.

When you are ready to test run `$ hexo server` and view your page in a browser. If you are happy `$hexo generate --deploy` and in a few seconds its online!

###Visual Studio Code
One of the coolest things about using Hexo to generate my blog is that I can curate the raw markdown files in my text editor of choice. No better excuse to load up [Visual Studio Code](https://code.visualstudio.com/). I'm loving the new tools that Microsoft are putting out. Code is a great text editor so far! And with the split screen and preview I can easily get a rough preview of what the post will look like!

{% asset_image "vscode.jpg" %}

###Thanks
So far I really like
So thanks to Kam for introducing me to Hexo, and thanks the Hexo team for creating a really great product. Now I just have to use it more often!

- Richard 