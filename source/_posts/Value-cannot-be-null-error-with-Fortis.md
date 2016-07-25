title: "Value cannot be null error with Fortis"
date: 2016-02-25 12:16:15
tags:
category: Fortis
---

I'm going to start posting about common issues that you might come across when using the [Fortis](http://fortis.ws) Wrapper for Sitecore.

This first one is something I see very regularly, and its usually a very simple problem to fix. 

```
Server Error in '/' Application.

Value cannot be null.
Parameter name: field

Description: An unhandled exception occurred.

Exception Details: System.ArgumentNullException: Value cannot be null.
Parameter name: field

Source Error:

An unhandled exception was generated during the execution of the current web request. 
Information regarding the origin and location of the exception can be identified using 
the exception stack trace below.

Stack Trace:


[ArgumentNullException: Value cannot be null.
Parameter name: field]
  Sitecore.Diagnostics.Assert.ArgumentNotNull(Object argument, String argumentName) +63
  Fortis.Model.Fields.FieldWrapper..ctor(Field field, ISpawnProvider spawnProvider) +37
```

If we start to look at the stack trace there is an `ArgumentNullException` thrown in the constructor of the `FieldWrapper` class. The parameter that is null is the `field` parameter. That is the `Sitecore.Data.Fields.Field` object that the Fortis `SpawnProvider` passes into the `FieldWrapper` to use as its internal field object. The `FieldWrapper` then "wraps" the Sitecore field object and provides properties and methods to get at the values.

The fact that the `Field` being passed in is null, usually means that when the Sitecore item was retrieved from the database, the field did not exist on that item.

###Things to check
* The first thing to check is that the field exists in the **master** database on the template that the item is based on. If not, you are probably missing something from the TDS/Unicorn/Sitecore Package or whatever method you use to sync up your databases. So get the field added and publish.

* If the field _does_ exist on the template, then it probably has not been published to the web database. Publish that template and you are good to go.

* Finally - maybe the field _should_ have been removed from the template. In that case, you haven't re-generated your Fortis model class since removing that field. If you re-gen the code, it will remove the reference to the field and your site will start working again.

Hopefully this helps someone!

Happy coding

-- Richard