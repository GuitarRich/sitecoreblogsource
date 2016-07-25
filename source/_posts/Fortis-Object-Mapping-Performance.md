title: "Fortis Object Mapping Performance"
date: 2015-06-11 12:35:04
tags:
- Sitecore
- Fortis
- Performance
category:
- Fortis
---
After the post on [Glass Mapper by Konstantin Cherkasov](http://sitecorepro.blogspot.com/2015/06/sitecore-and-glassmapper-how-much.html) performance, and the follow up by [Kam on Synthesis Object Mapping Performance](http://kamsar.net/index.php/2015/06/Synthesis-Object-Mapping-Performance/) I thought I’d run a quick test of [Fortis](http://Fortis.ws) in a similar situation. Konstantin used a GlassCast + field and found the access to be nearly 2,000 times slower than using the Sitecore API to access the same field value. Performance of Synthesis looked much better at only 3x slower than the Sitecore API. There will always be a small trade off in performance when using an ORM/Wrapper.

I tried to get a similar payload to Kam, my test consisted of 4716 items being pulled from the database, and then at that point I used the code from Kam's [Gist](https://gist.github.com/kamsar/f2aa92ef3f63f3c7b931), modified it to return a strongly typed collection of *IItemWrapper* objects, which is the base interface for all objects in fortis. Fortis is similar in concept to Synthesis in that it is a wrapper that reads from an internal Item instead of mapping to a POCO.

The results were very good - Fortis was not much slower at all than the native Sitecore API at all. These are the ranges of around 50 executions:
* Native: 6-12ms
* Fortis: 7-26ms
* Sample Size: 4716

For reference, this was Sitecore 8 Initial Release, running on a Quad Core i7, 16GB Ram, 512GB SSD. The [Gist for my test is here](https://gist.github.com/GuitarRich/7deb0f6eb689d8027c99) 

So great results for both [Systhesis](https://github.com/kamsar/Synthesis) and [Fortis](http://fortis.ws).