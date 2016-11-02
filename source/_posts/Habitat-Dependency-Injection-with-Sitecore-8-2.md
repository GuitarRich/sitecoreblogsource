---
title: Habitat Dependency Injection with Sitecore 8.2
tags:
  - Sitecore
  - Habitat
  - Helix
  - Dependency Injection
category: Sitecore
date: 2016-09-17 16:21:20
---

If you attended [@seanholmseby's](http://www.seanholmesby.com) session on making Habitat your home at Sitecore Symposium 2016, you will have heard him discuss dependency injection with Sitecore 8.2 and how you can use this in the [Habitat](https://github.com/Sitecore/Habitat) Demo project or a [Helix](http://helix.sitecore.net/) based project. One of the options discussed was to pass the container around in a pipeline and let each project register its own dependencies. This approach was also discussed by [Kevin Brechbühl](https://ctor.io/one-way-to-implement-dependency-injection-for-sitecore-habitat/) (@aquasonic for those on the slack chat). A new pipeline was created and called from the `Initialize` pipeline and the container was part of the pipeline args. While this approach does work it breaks some of the principles that govern good DI practice.

#### Composition Root
In Mark Seemann's book he describes the [*Composition Root*](http://blog.ploeh.dk/2011/07/28/CompositionRoot/) pattern. Most agree that we _should_ be using the *Constructor Injection* pattern when requiring dependencies, we have to also define an area that is responsible for composing those classes and their dependencies. Where should that be? Mark points out that this should be *As close as possible to the application's entry point*. This place is called the [*Composition Root*](http://blog.ploeh.dk/2011/07/28/CompositionRoot/) and is usually defined as:

> A Composition Root is a (preferably) unique location in an application where modules are composed together.

The problem with the approach discussed above is that it requires your library to reference the container. This breaks the Composition Root pattern which specifies:

> A DI Container should only be referenced from the Composition Root. All other modules should have no reference to the container.

And this makes sense - one of the purposes of [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection) is to remove tightly coupled dependencies. Adding in a dependency to the container in our libraries goes against that purpose.

#### Pre-Habitat/Helix
Pre-[Habitat](https://github.com/Sitecore/Habitat)/[Helix](http://helix.sitecore.net/) it was a fairly simple process to create a class in `App_Start` or if you wanted to keep things 'Sitecorey' (not the same as [@SiteCorey](https://twitter.com/sitecorey)!) you could add a new processor to the `Initialize` pipeline.

You _can_ still do this with Helix. But with the Helix design patterns, we have multiple features/projects that all have dependencies. Also - we need to be able to easily add new features, remove features without having to update the main application project each time. In fact, if you look at the website project file for Habitat - it does not contain references to _all_ the feature or foundation libraries.

Dependency Injection really fits in the Foundation layer just as in Kevin's example. So how can we do this, but _not_ add a dependency to the container in all the projects


#### Microsoft Dependency Injection Abstractions
With 8.2, Sitecore have embraced DI at last!

{% asset_img happydance.gif "Wooooooo" %}

I wont go into the details here. [@Kamsar](http://kamsar.net/index.php/2016/08/Dependency-Injection-in-Sitecore-8-2/) and [@Akshay](http://www.akshaysura.com/2016/09/15/microsoft-extensions-dependency-injection-di-with-sitecore-8-2-sample-project/) have already done that for you.

One of the features of the MS DI Abstractions is the new interface [`IServiceCollection`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/Extensions/DependencyInjection/IServiceCollection/) - this specifies the contract for a collection of service descriptors.

This will enable us to build a collection of our dependencies and services, without a direct reference to the container. Also the registrations are not created in our feature. The Sitecore application or our own pipeline can take care of that, obeying the *Composition Root* pattern.

So how does that work? We have a few options!

#### Option 1: Just use Sitecore's Container
The simplest method to register your dependencies is to just use Sitecore's own DI abstractions and container. Each foundation and feature project just needs a single class and some config. This has been covered in other posts, but for completeness here is the method. Lets update the Accounts Feature from the Habitat demo project.

First we need a configurator, this will add our registrations to the `IServiceCollection` for the application:

```csharp
namespace Sitecore.Feature.Accounts
{
    using Microsoft.Extensions.DependencyInjection;
    using Sitecore.DependencyInjection;
    using Sitecore.Foundation.DependencyInjection;

    public class RegisterDependencies : IServicesConfigurator
    {
        public void Configure(IServiceCollection serviceCollection)
        {
            serviceCollection.AddTransient<IAccountRepository, AccountRepository>();
            // TODO: Add any other registrations here

            serviceCollection.AddMvcControllersInCurrentAssembly();
        }
    }
}
```

Then we need to add the config for Sitecore to know about the configurator:
```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <services>
      <configurator type="Sitecore.Feature.Accounts.RegisterDependencies, Sitecore.Feature.Accounts" />
    </services>
  </sitecore>
</configuration>
```

So simple right? 

But why might we choose this option? The MS DI Container that Sitecore uses is fast, in [test's it is at least as fast as Simple Injector](http://www.akshaysura.com/2016/08/31/ioc-container-benchmark-comparison-2016-including-microsoft-extensions-dependencyinjection/). Also - it's Sitecore's officially supported container. Sitecore use this to test their own code and so you are more likely to get a good response from support if you are using the same container as they do.

Another really good reason is if you want to use any of the Sitecore registrations in your code. These might be `ISettings`, `IFactory`, `BasePublishManager` etc... There are a lot of them! They will all be available in the container.

But the conforming container used in the MS DI Abstractions does not have some advanced container features that you might want to use. So....

#### Option 2: Use your own Container
Many people already have DI setup in pre-8.2 projects. This could be using AutoFac, Simple Injector, Structure Map - there are a lot of options. So why keep your own container? Familiarity? Performance... or you might use some of the advanced container features that are not available in the MS DI container.

Also there is the argument that you should keep your framework and application containers separate. Whatever your reason, how can you do this and keep the composition root principle.

First we need to make sure that we register our dependencies and set the container _before_ Sitecore does its thing. Why? Here is the code for the Sitecore dependency resolver:

```csharp
  public class SitecoreDependencyResolver : IDependencyResolver
  {
    private readonly IDependencyResolver _innerResolver;

    public SitecoreDependencyResolver(IDependencyResolver innerResolver)
    {
      this._innerResolver = innerResolver;
    }

    public object GetService(Type serviceType)
    {
      return ServiceLocator.ServiceProvider.GetService(serviceType) ?? this._innerResolver.GetService(serviceType);
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
      IEnumerable<object> source = (IEnumerable<object>) ServiceLocator.ServiceProvider.GetService(typeof (IEnumerable<>).MakeGenericType(serviceType));
      object[] objArray = source as object[] ?? source.ToArray<object>();
      if (source != null && ((IEnumerable<object>) objArray).Any<object>())
        return (IEnumerable<object>) objArray;
      return this._innerResolver.GetServices(serviceType);
    }
  }
```

You can see that the current `DependencyResolver` is passed in and then stored as the `_innerResolver` - then if the Sitecore container cannot resolve the type, it falls back to the `_innerResolver`. This may have performance implications on your code, so you could write your own dependency resolver and fall back the other way if you prefer.

Now we can take Kevin's approach and modify it so that instead of passing the container in the args, we pass an `IServiceCollection` and use that to build our service descriptors.

The pipeline args now look like this:

```csharp
namespace Sitecore.Foundation.DependencyInjection.Pipelines.InitializeDepdencyInjection
{
  using Microsoft.Extensions.DependencyInjection;
  using Sitecore.Pipelines;

  public class InitializeDependencyInjectionArgs : PipelineArgs
  {
    public IServiceCollection ServiceCollection { get; set; }

    public InitializeDependencyInjectionArgs(IServiceCollection serviceCollection)
    {
      this.ServiceCollection = serviceCollection;
    }
  }
}
```

We have to modify the `initialize` processor to create a new `ServiceCollection` and pass it into the args. Then after the pipeline has finished running, we will take those service descriptors and register them with the container. In conforming containers you can do `container.Populate` and pass in the `IServiceCollecion`. Simple Injector is not conforming right now so we have to do this bit ourselves. Here is the processor:

```csharp
namespace Sitecore.Foundation.DependencyInjection.Pipelines.Initialize
{
  using System;
  using System.Collections.Generic;
  using System.Linq;
  using System.Web.Mvc;
  using Microsoft.Extensions.DependencyInjection;
  using SimpleInjector;
  using SimpleInjector.Integration.Web.Mvc;
  using Sitecore.Diagnostics;
  using Sitecore.Foundation.DependencyInjection.Pipelines.InitializeDepdencyInjection;
  using Sitecore.Pipelines;

  public class IntializeDepdencyInjection
  {
    public void Process(PipelineArgs args)
    {
      Log.Info("Start dependency injection initialization", this);

      var serviceCollection = new ServiceCollection();
      var container = new Container();

      // start the pipeline to register all dependencies
      var dependencyInjectionArgs = new InitializeDependencyInjectionArgs(serviceCollection);
      CorePipeline.Run("initializeDependencyInjection", dependencyInjectionArgs);

      var containerCache = new List<Type>();

      foreach (var serviceDescriptor in dependencyInjectionArgs.ServiceCollection)
      {
        // Safety check so we don't try to register the same type twice
        if (containerCache.Contains(serviceDescriptor.ServiceType))
        {
          continue;
        }

        Lifestyle siScope;
        switch (serviceDescriptor.Lifetime)
        {
          case ServiceLifetime.Singleton:
            siScope = Lifestyle.Singleton;
            break;

          case ServiceLifetime.Transient:
            siScope = Lifestyle.Transient;
            break;

          case ServiceLifetime.Scoped:
          default:
            siScope = Lifestyle.Scoped;
            break;
        }

        container.Register(serviceDescriptor.ServiceType, serviceDescriptor.ImplementationType, siScope);
        containerCache.Add(serviceDescriptor.ServiceType);
      }

      // Verify our registrations
      container.Verify();

      // Set the ASP.NET dependency resolver
      DependencyResolver.SetResolver(new SimpleInjectorDependencyResolver(container));
    }
  }
}
```

Finally we have to register the processor:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <pipelines>
      <initialize>
        <processor type="Sitecore.Foundation.DependencyInjection.Pipelines.Initialize.InitializeDependencyInjection, Sitecore.Foundation.DependencyInjection"
                patch:before="processor[@type='Sitecore.Mvc.Pipelines.Loader.InitializeControllerFactory, Sitecore.Mvc']" />
      </initialize>
    </pipelines>
  </sitecore>
</configuration>
```

#### Configure your dependencies
Now DI is setup for our own container, but the features need to configure thier own dependencies. So lets look at the Accounts module again and see how our configurator changes:

```csharp
public class RegisterServices
{
    public void Process(InitializeDependencyInjectionArgs args)
    {
        args.ServiceCollection.AddTransient<IAccountRepository, AccountRepository>();
        // TODO: Add any other registrations here

        args.ServiceCollection.AddMvcControllersInCurrentAssembly();
    }
}
```

and our configuration changes to add the processor to the pipeline:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
  <sitecore>
    <pipelines>
      <initializeDependencyInjection>
        <processor type="Sitecore.Feature.Accounts.Pipelines.InitializeDependencyInjection.RegisterServices, Sitecore.Feature.Accounts" />
      </initializeDependencyInjection>
    </pipelines>
  </sitecore>
</configuration>
```

#### Conclusion
With the new DI features in 8.2, it is possible to build a Helix based site and still obey good DI principles. While we are adding a dependency on the MS DI Abstractions, we are not adding a dependency on the container and we are still registering as close to the application's entry point as we can. At the same time we are keeping the component/modular based architecture of Helix intact and unpoluted.

So which option should you choose? Well I can't make that descision for you, I think there are pro's and con's for both options presented here. There is also a 3rd option that I haven't gone into where you can replace Sitecore's container for another conforming container.... if you really really wanted to do that, it is an option. For me I think the safe option is to just use Sitecore's own resolver/container. It is fast and makes the configuration simple. But if you want to use advanced features like Simple Injectors [`.Verify()`](http://simpleinjector.readthedocs.io/en/latest/howto.html#verify-the-container-s-configuration) method (which btw has saved my bacon many times!) - option 2 is a good alternative.

I would love to hear your comments, suggestions on what you think or how this could be improved. Just add below or give me a shout on the [Sitecore Slack](http://sitecorechat.slack.com) channels - I'm @guitarrich. All the code for option 2 can be found on my fork of the habitat source: [https://github.com/GuitarRich/Habitat/tree/feature/dependency-injetion-8.2](https://github.com/GuitarRich/Habitat/tree/feature/dependency-injetion-8.2).

{% asset_img guitarrich.jpg "@guitarrich" %}