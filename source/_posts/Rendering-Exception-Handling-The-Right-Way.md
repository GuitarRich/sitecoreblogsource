title: "Rendering Exception Handling - The Right Way"
date: 2015-10-23 12:02:08
tags:
- Sitecore
- Renderings
- Exceptions
- MVC
category: Sitecore
---

I see a few different ways of handling errors and error pages in Sitecore. I really like the way that we do it at [Lightmaker](http://www.lightmaker.com), it makes things nice and simple and easy to debug! It also allows us as developers to follow good practice in our business and data layers and not worry about exception handling.

### Don't Catch Exceptions!
[CA1031: Do not catch general exception types](https://msdn.microsoft.com/en-us/library/ms182137.aspx) - Best practice for exception handling says that we should never catch general exception types! If you have this in your code, its time to think again and refactor:

```cs
try
{
	// Do something 
}
catch(Exception ex)
{
	Log.Error("message", this, ex);
}
```

#### [The first rule of exception handling is: Do not do exception handling!](http://mikehadlow.blogspot.com/2009/08/first-rule-of-exception-handling-do-not.html)

> Catching `System.Exception` is the worst possible exception handling strategy. What happens when the application continues executing? It’s probably now in an inconsistent state that will cause extremely hard to debug problems in some unrelated place, or even worse, inconsistent corrupt data that will live on in the database for years to come.

It is so much better to simply allow any exceptins to bubble up to the top of the stack and leave a clear message and stack trace in the error log. Depending on the application, give some indication that there has been a problem in the UI.

### How can we do this with Sitecore
In your Sitecore implementation, if you are following a good componentised practice, you probably have a few layers in your code. This is an example of what you might have in an MVC Sitecore project:

**Business Layer**: Handles getting data from Sitecore for searches or other complex functionality.

**Controllers**: Communicates with the business layer or Sitecore API to get build a view model and pass it to the view.

**Presentation**: Views, razor scripts that mix the markup and the data from the view model and present it to the user.

At any point in this flow of data from the Sitecore database to the user and exception could occur. I'm sure you are all good developers and make sure that objects are not null before using them etc... But there are just something that are out of our control!

Also some parts of our code might need to throw an exception. What if a required field has not been filled in by an editor. Somehow, despite validation, something has gone wrong. In these cases we *should* throw an exception, something *has* gone wrong and it should be logged and someone notified.

### Where should we handle exceptions
The best thing would be a global exception handler at the application level, that catches any unhandled exception that our application throws. In Sitecore MVC we can do this by overriding the `ExecuteRenderer` processor in the `mvc.renderRendering` pipeline. In the process method, we can wrap the `base.Process` method with a try/catch and then handle any exceptions that occur.

```cs
public override void Process(RenderRenderingArgs args)
{
	try
	{
		base.Process(args);
	}
	catch (Exception ex)
	{
		args.Cacheable = false;

		var parametersException = GetRenderingParametersException(ex);
		if (parametersException != null)
		{
			this.RenderParametersError(args, parametersException);
			Log.Error(parametersException.Message, parametersException, this);
		}
		else
		{
			this.RenderError(args, ex);
			Log.Error(ex.Message, ex, this);
		}
	}
}
```

Create this include file to override the processor:

```xml
<?xml version="1.0"?>
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<pipelines>
			<mvc.renderRendering>
				<processor type="Sitecore.Mvc.Pipelines.Response.RenderRendering.ExecuteRenderer, Sitecore.Mvc">
					<patch:attribute name="type">Sitecore.Exception.Handler.Pipelines.Response.RenderRendering.RenderRendering.ExecuteRenderer, Sitecore.Exception.Handler</patch:attribute>
				</processor>
			</mvc.renderRendering>
		</pipelines>
	</sitecore>
</configuration>
```

If we catch an exception, we can log the error and also render a custom view in place of the original view for the rendering. Doing this means that we can display meaningful error messages for predefined conditions. A standard setup at Lightmaker is to display the error message if we are in the Page/Experience editor or if `customErrors mode="Off"`

To render the error view, we render the view to a string and use the `RenderRenderingsArgs.Writer` to output the reusult:

```cs
private void RenderError(RenderRenderingArgs args, Exception ex)
{
	args.Writer.Write(this.RenderViewToString("Errors/RenderingError", GetRenderingErrorModel(args, ex), GetControllerContext(args)));
}
```

RenderViewToString:

```cs
public string RenderViewToString(string viewName, object model, ControllerContext controllerContext)
{
	if (string.IsNullOrEmpty(viewName))
	{
		viewName = controllerContext.RouteData.GetRequiredString("action");
	}

	controllerContext.Controller.ViewData.Model = model;

	using (var stringWriter = new StringWriter())
	{
		var viewEngineResult = ViewEngines.Engines.FindView(controllerContext, viewName, null);

		var viewContext = new ViewContext(
			controllerContext,
			viewEngineResult.View,
			controllerContext.Controller.ViewData,
			controllerContext.Controller.TempData,
			stringWriter);

		viewEngineResult.View.Render(viewContext, stringWriter);

		return stringWriter.GetStringBuilder().ToString();
	}
}
```

This means that in our controllers, we can throw a `RenderingParameterException` or a `DatasourceNotSet` exception if the datasource or rendering parameters have not been set correctly. In the page/experience editor, the user would get a message indicating that the data needs to be fixed. But on the delivery website, the error is hidden from the end users.

Also when debugging the site locally or in a test/qa environment - where we can set custom errors to `Off`, we can see the errors being caught.

The razor view for a rendering error would look like this:

```razor
@model Sitecore.Exception.Handler.Models.RenderingErrorModel
         
@if (!Sitecore.Context.PageMode.IsNormal || !HttpContext.Current.IsCustomErrorEnabled)
{
	<div class="error-messages">
		<h4 class="error-header">
			Error while rendering the view [@Model.RenderingName] Please, make sure the rendering is configured properly or contact your administrator.
		</h4>
		<ul>
			@foreach (var line in Model.Exception.ToString().Split(new[] { Environment.NewLine }, StringSplitOptions.RemoveEmptyEntries).Select(x => x.Trim()))
			{
				<li @(line.StartsWith("at") ? "style=padding-left:40px;" : string.Empty)>@line</li>
			}
		</ul>
	</div>
}
else
{
	<!-- Error while rendering the view [@Model.RenderingName] - [@Model.Exception] -->
}
```

For full details of the code, including the methods that are referenced but not listed here - please see the project at [https://github.com/GuitarRich/sitecore-exception-handler](https://github.com/GuitarRich/sitecore-exception-handler). This is currently being made into an installable Sitecore package and will be submitted to the Market Place this week.

Happy coding!

--Richard