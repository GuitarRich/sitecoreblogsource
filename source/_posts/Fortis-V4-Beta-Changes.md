title: "Fortis V4 Beta Changes"
date: 2016-03-28 15:45:19
tags:
category: Fortis
---

We are working on new version of [Fortis](http://fortis.ws) so I'm going to do a few posts to highlight the new improvements. If you have any feature requests please make sure they are added as an issue/request on Github: [https://github.com/Fortis-Collection/fortis/issues](https://github.com/Fortis-Collection/fortis/issues).

###Supporting Habitat
A breaking change to Fortis is how the configuration is handled. Previously, the core Fortis package modified the `web.config` to add a section that allowed the developer to identify the assembley that the Fortis model classes were held in. This limited the development team to a single assembly for all the models. 

```xml
<configuration>
    <configSections>
        <section name="fortis" type="System.Configuration.NameValueSectionHandler" />
    </configSections>
    <fortis>
        <add key="assembly" value="LM.Model, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
    </fortis>
</configuration>
```

This was limiting to the development team. It meant that you could not build a module using Fortis and install that to a Sitecore implementation without serializing the templates for the module in your site and having the model classes for the module in your own project.

Recently the big talking point has been [Sitecore Habitat](https://github.com/Sitecore/Habitat) and how that is archtected into Features/Components. Again, having a single assembly forces all the model classes and interfaces to live in the same project, breaking the architectural pattern.

In Fortis V4 this has all been changed. To be more aligned with standard Sitecore modules, we have moved the configuration out of the `Web.config` and into an include file. This file lives by default in `/app_config/include/fortis/fortis.config`.

Also the models can now live in multiple assemblies. This is done by adding each assembly to the `models` list in the config. Because this is now inside the Sitecore config, you can do this via an include file for each of your modules:

```xml
<models>
    <model name="Fortis.Model" assembley="Fortis.Model, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
    <model name="MyFeature.Model" assembley="MyFeature.Model, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
</models>
```

Patch example:
```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
    <sitecore>
        <fortis>
            <models>
                <model patch:after="*[name='Fortis.Model']" name="MyFeature.Model" assembley="MyFeature.Model, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            </models>
        </fortis>
    </sitecore>
<configuration>
```

###Custom Field Types
Another bug bear has been dealing with custom field types in Fortis. Previously the field type support was hard coded into the `SpawnProvider`. To support a custom field type you would have to roll your own `SpawnProvider` and implement the `GetField` method for your field type.

This has now been moved to the configuration:

```xml
<fieldTypes>
    <field name="checkbox" type="Fortis.Model.Fields.BooleanFieldWrapper, Fortis" />
    <field name="image" type="Fortis.Model.Fields.ImageFieldWrapper, Fortis" />
    <field name="date" type="Fortis.Model.Fields.DateTimeFieldWrapper, Fortis" />
    <field name="datetime" type="Fortis.Model.Fields.DateTimeFieldWrapper, Fortis" />
    <field name="checklist" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="treelist" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="treelistex" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="multilist" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="multilist with search" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="treelist with search" type="Fortis.Model.Fields.ListFieldWrapper, Fortis" />
    <field name="file" type="Fortis.Model.Fields.FileFieldWrapper, Fortis" />
    <field name="droplink" type="Fortis.Model.Fields.LinkFieldWrapper, Fortis" />
    <field name="droptree" type="Fortis.Model.Fields.LinkFieldWrapper, Fortis" />
    <field name="general link" type="Fortis.Model.Fields.GeneralLinkFieldWrapper, Fortis" />
    <field name="general link with search" type="Fortis.Model.Fields.GeneralLinkFieldWrapper, Fortis" />
    <field name="text" type="Fortis.Model.Fields.TextFieldWrapper, Fortis" />
    <field name="single-line text" type="Fortis.Model.Fields.TextFieldWrapper, Fortis" />
    <field name="multi-line text" type="Fortis.Model.Fields.TextFieldWrapper, Fortis" />
    <field name="droplist" type="Fortis.Model.Fields.TextFieldWrapper, Fortis" />
    <field name="rich text" type="Fortis.Model.Fields.RichTextFieldWrapper, Fortis" />
    <field name="number" type="Fortis.Model.Fields.NumberFieldWrapper, Fortis" />
    <field name="integer" type="Fortis.Model.Fields.IntegerFieldWrapper, Fortis" />
    <field name="name value list" type="Fortis.Model.Fields.NameValueListFieldWrapper, Fortis" />
</fieldTypes>
```

Now its as simple as adding a new `field` to the config to support a custom field type. Again, this can be patched in via an include file. If the custom field type stores its Raw Value in the same way as one of the in built fields, you can just use one of the existing `IFieldWrapper` classes. Or you can roll your own field wrapper as long as it implements `IFieldWrapper`.


###Other changes

Issues from GitHub:
* [#23](https://github.com/Fortis-Collection/fortis/issues/23): The `.Value` property on the `TextFieldWrapper` now returns the raw value as a string rather than running through the `FieldRenderer`. The `RichTextFieldWrapper.Value` property has been changed to return an `IHtmlString` and renders the rich text content with `Editing` set to false.
* [#19](https://github.com/Fortis-Collection/fortis/issues/19): The `IItemFactory.Save` method has new overloads to match the `.Save(bool silent)` and `.Save(bool updateStatistics, bool silent)`.
* [#18](https://github.com/Fortis-Collection/fortis/issues/18): Added a new `.HasValue` property that can be used to check if a value is blank or has not been assigned.


###Unit Testing Changes
Finally we have made some changes to the unit tests. Previously the unit tests had been created using Microsoft Fakes, but this relied on the developer having a Visual Studio Ultimate edition or higher. These tests have been removed in favor of using [Sitecore FakeDb](https://github.com/sergeyshushlyapin/Sitecore.FakeDb). This will make the unit tests more accessible to developers that do not have access to the ultimate editions of VS. More tests will be ported to the new version in the coming weeks.

This has now been released on Nuget as [v4.0.0-beta3](https://www.nuget.org/packages/Fortis/4.0.0-beta3) - have a play and let us know what you think!

Well thats all for now - there will be more updates coming soon, please submit issues or feature requests on Github or hit us up on Slack!

-- Richard.
