title: "Fortis Model Generation with Unicorn 3"
date: 2015-10-14 17:38:38
tags:
category: Fortis
---


After using Unicorn 2.x for a while now, I was excited at the prospect of [Unicorn 3](http://kamsar.net/index.php/2015/10/Unicorn-3-0-Released/) - with its new serialization format that solves a lot of the problems with the standard Sitecore serialization API (*cough* long file names *cough*)! I can also confirm that the speed increases are pretty dramatic, I believe the claims of 50% faster than Unicorn 2!

{% asset_img Unicorn_logo.png "Unicorn 3" %}

The new serialization format caused a small problem tho. With [Fortis](http://fortis.ws), all the model classes are auto generated using a T4 template. The current code uses the [Transitus](http://fortis.ws/fortis-collection/transitus/) NuGet package to read those files into strongly typed objects and then the T4 template iterates through those objects to build the interfaces and classes for our Sitecore models.

This didn't work as Transitus uses the Sitecore serialization API to read the files.

###Rainbow API
With some pointers from [Kamsar](http://kamsar.net) and digging around the [Rainbow](https://github.com/kamsar/Rainbow) source code - I found I could just use the [YamlSerializationFormatter.cs](https://github.com/kamsar/Rainbow/blob/master/src/Rainbow.Storage.Yaml/YamlSerializationFormatter.cs) to read the YAML files and deserialize them into the same strongly typed objects that were used in Transitus.

Just pass in the filename and the `RainbowItem` object is returned:

```
public IItem ReadItem(string filePath)
{
	var file = new FileInfo(filePath);

	lock (FileUtil.GetFileLock(file.FullName))
	{
		using (var reader = file.OpenRead())
		{
			var readItem = this.SerializationFormatter.ReadSerializedItem(reader, filePath);
			readItem.DatabaseName = "master";
			return new RainbowItem(readItem);					
		}
	}
}
```

###Transitus.Rainbow
So [Transitus.Rainbow](https://www.nuget.org/packages/Transitus.Rainbow/) was born. This is available for download now from [Nuget](http://nuget.org).

To use, add `Transitus.Rainbow` to your project using the Nuget package manager or entering the following command in the Package Manager Console:

```
PM> Install-Package Transitus.Rainbow
```
This should install the `Rainbow.Core` and `Rainbow.Storage.Yaml` packages also.  

Now you can use the `TransitusProvider.FolderDeserializer` to get all the items deserialized into useable objects:

```
var items = Transitus.Rainbow.TransitusProvider.FolderDeserializer.Deserialize(@"C:\Projects\transitus.rainbow\Files");
var templates = Transitus.Rainbow.TransitusProvider.TemplateFactory.Create(items);
```

These template items can then be used to generate the code.

###Fortis.Model Project
To see this in use you can clone this repo: [https://github.com/Fortis-Collection/fortis.codegen.transitus.rainbow](https://github.com/Fortis-Collection/fortis.codegen.transitus.rainbow).

This project is a sample of how to generate the Fortis models:

* `AssembleyReferences.tt` : This template contains the refernces needed by the `Fortis.t4` template. Is is added to the includes.
* `Fortis/Fortis.t4` : This is the main code generation template. It gets all the files from the folders supplied by `Model.tt`, gets the templates available in the files and generates an interface and implementation for each Sitecore template.
* `Model.tt` : This file sets up all the folders and the base settings for the `Fortis.t4` template to use. This file should be customized for each project.

Please checkout the demo repo and give me or [Jason Bert](http://www.jasonbert.com/) a shout with any questions!

-- Richard