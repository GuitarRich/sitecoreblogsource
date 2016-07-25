title: "Unit Testing with Sitecore and Fortis - Part 1"
date: 2014-11-11 09:35:41
tags:
- Sitecore
- Unit Testing
- Fortis
category: Fortis
---

This post is the start of a series of articles looking at unit testing using [Sitecore](http://www.sitecore.net/) and [Fortis](http://fortis.ws). There are already a lot of great posts on unit testing with Sitecore using fakes and other tools, so we are going to primarily focus on how to setup unit testing when using the Fortis framework. *We will go into some of the other benefits of using Fortis in a future post.*

One of the really useful things about Fortis is how it abstracts away the Sitecore API and models your data templates. All this is done via the **IItemFactory** and **IItemWrapper** intefaces, allowing you to easily unit test your code logic without having to worry about mocking/faking Sitecore.

In this post we will discuss how to setup the unit tests and give some simple examples. In following posts we will go more in depth and cover unit tests for more complex code.

### Background

Here at Lightmaker we group our components logically by function rather than by type. Each component consists of one or more of the following:

*   **Rendering**: A view or controller rendering that holds the presentation elements
*   **Controller**: A simple controller action that requests the data from a manager class and sends the model to the view/rendering
*   **Manager**: The object that does the heavy lifting and logic. This is the main part of our code that gets unit tested
*   **Rendering Model**: The model for the view. This always inherits from Fortis.Models.IRenderingModel so that all views have access to the PageItem, the RenderingItem and the RenderingParametersItem<sup>*</sup>

### Example
So let’s look at a code example. Let’s take a component that display’s frequently asked questions. For this we have the following items created in Sitecore:

*   **Templates:**
    *   **Frequently Asked Question**: This template contains 2 fields, Question [a single-line text field] and Answer [Rich Text]
    *   **Frequently Asked Questions Folder**: Inherits from the system/common/folder template, has insert options set to Frequently Asked Question.
    *   **Content Page**: A generic page type that contains the scaffolding for site pages.
*   **Renderings:**
    *   **Frequently Asked Question**: A simple controller rendering
*   **Managers:**
    *   **IFAQManager** : The main manager class to handle the business logic

### Project Setup

Here is the basic project setup:
{% asset_img Project-Setup.jpg "Project Setup" %}

### The Code

The IFAQManager contains a single method – **GetFAQsFromParent** – that takes an **IFrequentlyAskedQuestionsFolder** object and return an **IEnumerable**.

The interface and implementation:

``` csharp
public interface IFAQManager
{
	IEnumerable GetFAQsFromParent(IFrequentlyAskedQuestionsFolder parentFolder);
}

public class FAQManager : IFAQManager
{
	public IEnumerable GetFAQsFromParent(IFrequentlyAskedQuestionsFolder parentFolder)
	{
		var childItems = parentFolder.Children().OrderBy(x => x.Name);
		return childItems;
	}
}
```

The controller:

``` csharp
public class FAQController : BaseController
{
	protected readonly IFAQManager FAQManager;

	public FAQController(ICustomItemFactory itemFactory, ILoggingManager log, IFAQManager faqManager)
		: base(itemFactory, log)
	{
		this.FAQManager = faqManager;
	}

    public ActionResult FAQList()
    {
        var model =
	        new FAQRenderingModel(
		        this.ItemFactory
			        .GetRenderingContextItems<IContentPage, IFrequentlyAskedQuestionsFolder, IFAQRenderingParameters>(
				        this.ItemFactory));

		if (model.RenderingItem == null)
		{
			throw new RenderingParametersException("Data source");
		}

		model.FrequentlyAskedQuestions = this.FAQManager.GetFAQsFromParent(model.RenderingItem);

		return this.View(model);
    }
}
```

The model:
``` csharp
public class FAQRenderingModel : RenderingModel<IContentPage, IFrequentlyAskedQuestionsFolder, IFAQRenderingParameters>
{
	public FAQRenderingModel(IRenderingModel<IContentPage, IFrequentlyAskedQuestionsFolder, IFAQRenderingParameters> model)
		: base(model.PageItem, model.RenderingItem, model.RenderingParametersItem, model.Factory)
	{
	}

	public IEnumerable<IFrequentlyAskedQuestion> FrequentlyAskedQuestions { get; set; }
}
```

### The Unit Test

We will be using [Moq](https://github.com/Moq/moq4 "Moq") and [NUnit](http://www.nunit.org/ "NUnit") to complete the testing. Here is the code for the Unit Test:

``` csharp
[TestFixture]
public class FAQManagerTests
{
	[Test]
	public void GetFAQsFromParentTest()
	{
		// Create a mock of the datasource item
		var dataSource = new Mock<IFrequentlyAskedQuestionsFolder>();

		// Create some test data
		var faq1 = new Mock<IFrequentlyAskedQuestion>();
		var faq2 = new Mock<IFrequentlyAskedQuestion>();
		var faq3 = new Mock<IFrequentlyAskedQuestion>();
		var faq4 = new Mock<IFrequentlyAskedQuestion>();

		var faqList = new List<IFrequentlyAskedQuestion> { faq1.Object, faq2.Object, faq3.Object, faq4.Object };

		// Setup the datasource to return the child items.
		dataSource.Setup(x => x.Children<IFrequentlyAskedQuestion>(recursive: false)).Returns(faqList);

		// Create the FAQ Manager
		var faqManager = new FAQManager();

		var returnedItems = faqManager.GetFAQsFromParent(dataSource);
		Assert.AreEqual(faqList.Count, returnedItems.Count());
	}
}
```

This is a very basic example of how to unit test with Fortis. In the next posts in this series, we will breakdown whats happening here and look at some more complex test cases, including testing the controllers.

<sup>*</sup> The Fortis rendering model has the following properties:

*   **PageItem**: A model of the current Sitecore context item. Equivalent of Sitecore.Context.Item
*   **RenderingItem**: A model of the item set by the renderings DataSource
*   **RenderingParametersItem**: A model of the rendering parameters associated with the current rendering
