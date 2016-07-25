title: "Simple Injector and WFFM Controller Injection Woes"
date: 2015-07-27 15:33:32
tags:
- Sitecore
- IoC
- WFFM
category: Sitecore
---

We had a client recently upgrade to Sitecore 8 Update 4, who used both Ninject and Web Forms For Marketers. During the upgrade, we also migrated the Ninject code over to Simple Injector as the performance benefits have been proven. [IoC Container Performance – The impact on Sitecore](http://cardinalcore.co.uk/2015/02/12/ioc-container-performance-the-impact-on-sitecore/) and [IoC container benchmarks and features](http://www.palmmedia.de/Blog/2011/8/30/ioc-container-benchmark-performance-comparison).

But after we did this, the Form Reports stopped reporting data. After some investigation it was clear that the data was being written to both the MongoDB and the SQL Reporting databases. On closer inspection on the reports form, the ajax calls being made to retrive the data were throwing errors.

{% asset_img WFFM-Server-Error.jpg %}

The WFFM `FormsController` has 2 constructors and out of the box Simple Injector does now allow that. One of the Simple Instructor desgin decisions is [Allow only a single constructor](https://simpleinjector.readthedocs.org/en/latest/decisions.html#allow-only-a-single-constructor). In most cases that is a great thing, all the code at Lightmaker follows that standard anyway. But this Controller is part of the WFFM code base, so we are limited in our options. 

Looking at the 2 constructors we can see that one has an `IWFFMDataProvider` injected in, but the default one populates that object from the config. It looks like that is the one we want to use. So how to get Simple Injector to ignore the 2nd constructor?

``` csharp
public class FormController : SitecoreController, IHasModelFactory, IHasFormDataManager
{
	public IModelFactory ModelFactory { get; private set; }
	
	public IFormDataManager FormManager { get; private set; }
	
	public FormController()
		: this((IModelFactory) Factory.CreateObject("wffm/modelFactory", true), 
			(IFormDataManager) Factory.CreateObject("wffm/formDataManager", true))
	{
	}
	
	public FormController(IModelFactory modelFactory, IFormDataManager formDataManager)
	{
		Assert.ArgumentNotNull((object) modelFactory, "modelFactory");
		Assert.ArgumentNotNull((object) formDataManager, "formDataManager");
		this.ModelFactory = modelFactory;
		this.FormManager = formDataManager;
	}
}
```

### Constructor Behavior Options
Fortunately Simple Injector can be extended to cope with this. We can use a custom `ConstructorResolutionBehavior` class to "tell" Simple Injector which constructor to use.

Here is the resolution:

``` csharp
public class DefaultFirstConstructorBehavior : IConstructorResolutionBehavior
{
	public ConstructorInfo GetConstructor(Type serviceType, Type implementationType)
	{
		var constructors = implementationType.GetConstructors();
		if (constructors.Any())
		{
			return (
				from ctor in constructors
				orderby ctor.GetParameters().Length ascending 
				select ctor)
				.First();
		}
		
		return null;
	}
}
```

And set this when you create the container:
``` csharp
var container = new Container();
container.Options.ConstructorResolutionBehavior = new DefaultFirstConstructorBehavior();
```

Now, Simple Injector selects the default constructor for WFFM FormController actions and the reports are showing data again. I feel that we could make this a little safer by explicitly stating the controllers that get the default constructor. But for now in our project this works nicely. There hasn't been any ill effects on any other part of the CMS environment yet!

-- Richard.