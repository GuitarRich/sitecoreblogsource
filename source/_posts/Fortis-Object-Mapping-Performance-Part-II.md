title: "Fortis Object Mapping Performance - Part II"
date: 2015-06-15 10:15:03
tags:
- Fortis
- Sitecore
- ORM
category:
- Fortis
---

Last week I wrote a blog on Fortis Object Mapping Performance, I have a few updates!

Kam pointed out a flaw in my original code. I populated an array of Fortis objects, but I did not read out a field as Kam had done in his code. So the comparison was not fair. I have now updated my [Gist](https://gist.github.com/GuitarRich/7deb0f6eb689d8027c99) to read out the Display name. As expected, this does add a hit to performance, but Fortis still performs very well and in the real world you would not really notice the hit by using [Fortis](http://fortis.ws).

The new performance figures after running the code 100 times are:

* Native: 5-12ms
* Fortis: 16-32ms
* Sample Size: 4716

So we are really happy with the object mapping performance on Fortis.

Also there was a follow up on the Glass Performance by Nat - [Life Through a Lens – ORM Mythology & Glass Mapper Relative Performance](http://cardinalcore.co.uk/2015/06/15/life-through-a-lens-glass-mapper-relative-performance/). He makes some very good points, and also fixed the test code where it now has much better performance.

### Are these tests Usefull?
My favourite quote from Nats blog:

> In truth – I would shoot a developer that attempted to manipulate 20,000 records worth of data through an ORM – just plain silly ;)

I completely agree on this. In the real world we would rarely be working with that number of items (Maybe in bulk data imports/exports etc...). But when dealing with websites that have page views in the millions per day rather than hunderds/thousands, milliseconds do make a big difference. So I do find it interesting on the performance figures.

True, in those cases for large volume websites - Caching plays a much bigger part in performance. Hopefully this week I will get to finish off my blog post on output caching and get the shared source module up on the [Market Place](http://marketplace.sitecore.net)

 