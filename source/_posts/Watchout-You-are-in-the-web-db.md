title: "Watchout! You are in the web db!"
date: 2016-05-27 18:51:25
tags:
category: Sitecore
---

Based on a vent by [@cassidydotdk](https://twitter.com/cassidydotdk) on the [Sitecore Slack chat](http://sitecorechat.slack.com) the other day, we were all laughing about the (multiple)times when we forget that we have switched to teh **web** database to check something out, then forgot to switch back and edited some content. Only to find out that the content reverts when we publish... cos we are in the WEB db dummy!

It has happened to me more times than I care to mention!

###Solutions
A few solutions are available: There is a nice Google Chrome plugin: [Sitecore Extensions](https://alan-null.github.io/2016/05/sitecore-extensions_v1_0_0). This puts the DB name in the title bar of the Sitecore window. Makes it bright red if you are in the WEB database.

{% asset_img SitecoreExtensionsPlugin.png "I'm safe and in the master db!" }

{% asset_img SitecoreExtensionsPluginWebDB.png "Danger Will Robinson! You are in the Web db" }


That seems to work nicely, but only for Chrome users.

Then [@kam](https://twitter.com/kamsar) posted a quick content editor warning code snippet that works nicely on all browsers. But requires coding, deployment etc...

###Gotta be able to do that in SPE!!
So I wondered if we can do that quicker in [Sitecore PowerShell Extensions](https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx). Turns out its really easy to create content editor warngings and other content editor goodies with SPE. If you have never installed SPE and had a play with it, you need to do it now. It is easily the best developer module you can get for Sitecore, it *will* make your life better :)

####Create a new module
Let's start by creating a new PowerShell module. Head to the `/sitecore/system/Modules/PowerShell/Script Library`, right click and select **Module**:
{% asset_img CreateNewModule.png "Create new PowerShell Module" %}


This actually runs a wizard (written in SPE btw) that guides you through setting up a new SPE Module.

First give your module a name, then select the integration points to create. SPE can hook into a lot of area's inside Sitecore, for this module we just want to select `Content Editor Warning`:
{% asset_img CreateNewModule-Step2.png "Select the integration points" %}


Then click *Proceed*. Once the module is created, you can select your new item in the content tree and update some of the details in the about section.

{% asset_img CreateNewModule-Step3.png "Enable your module and add some details" %}


Next, open up the tree until you see the **Warning** item, this is where you will add your PowerShell script items. Right click and create a new PowerShell script:

{% asset_img CreateNewModule-Step4.png "Create the script" %}


Add a PowerShell Script item and load it up using PowerShell ISE. This is where the magic will happen! Because the SPE integration for content editor warning hooks into the `getContentEditorWarnings` pipeline, to set the warning title, text and icon in PowerShell we can just use the pipeline args object. That is passed into the PowerShell function for us:

```PowerShell
$title = "CE Warning Title"
$text = "Some Text"
$icon = @{$true="Office/32x32/information.png";$false="Applications/16x16/warning.png"}[$SitecoreVersion.Major -gt 7]

$warning = $pipelineArgs.Add($title, $text);
$warning.Icon = $icon
```

Now we need to get the database for the current item being edited. Because we are in a pipeline `pipelineArgs.Item` gives us that. So lets check the `item.Database.Name` and if we are not in the master, show the warning:

```PowerShell
$item = $pipelineArgs.Item
$databaseName = $item.Database.Name

if ($databaseName -eq "master") {
    return
}

$title = "WARNING: YOU ARE NOT IN THE MASTER DATABASE!!!"
$text = "You are currently editing this item while in the [${databaseName}]! You should only do this if you really know what you are doing! Do you know what you are doing? Will you be my friend?"
$icon = @{$true="Office/32x32/warning.png";$false="Applications/16x16/information.png"}[$SitecoreVersion.Major -gt 7]

$warning = $pipelineArgs.Add($title, $text)
$warning.Icon = $icon
```

And there you have it. Now when in the web or core database, a content editor warning appears at the top of the editor window:

{% asset_img ContentEditorWarning.png "Content Editor Warning" %}

To save you some time, here is a package file with the PowerShell Module: {% asset_link WhatDbAreYouInWarning.zip "What db are you in SPE Module" %}