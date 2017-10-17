---
title: 'Sitecore 9: Upgrading to Dynamic Placeholders'
tags:
  - Sitecore 9
  - Sitecore
  - Dynamic Placeholders
category: Sitecore 9
date: 2017-10-17 08:00:00
---


{% asset_img upgrade.jpg "Just do the upgrade!" %}

So with the new Dynamic Placeholders implementation finally here, how can we go about upgrading. Well, a lot depends on which implementation of Dynamic Placeholders you have used. In this example, we will assume that you have used the [Fortis](https://github.com/Fortis-Collection/dynamic-placeholders) module. But the principles used here can also be applied to other implementations.

## What are the differences

First, we want to see the differences between what Sitecore outputs and what Fortis does.

The Fortis placeholder generation uses `placeholderKey_renderingId` as the pattern to generate the key. This gives you keys like this:

```text
column_cdff3b6c-c0a7-4d6c-9a48-da82d186804c
```

The Sitecore implementation is very similar, but formats the Guid differently and also adds a unique key like this

```text
column_{CDFF3B6C-C0A7-4D6C-9A48-DA82D186804C}_0
```

## Updating the module

When dealing with the project, the first thing to do will be to remove the Dynamic Placeholder binary and config from your solution. You can uninstall the nuget package for this.

In your deployed instance, you will need to make sure that the following files are removed:

* `\App_Config\Include\DynamicPlaceholders\DynamicPlaceholders.config`
* `\bin\DynamicPlaceholders.dll`

## Updating the code

Once those files have been removed we need to make sure that all code referencing the namespace is changed to the new Sitecore namespace. So a search and remove this namespace: `DynamicPlaceholders.Mvc.Extensions` in your code. If you have used `@Html.Sitecore()` in your razor views, you probably already have the namespace set correctly. If not, make sure you have these namespaces added to your views `web.config`:

```xml
<add namespace="Sitecore.Mvc"/>
<add namespace="Sitecore.Mvc.Presentation"/>
```

Once we have done that, most of the code should work in the razor views. Unless you have used the overload and passed in a unique key to the Fortis ones. All the calls to `@Html.Sitecore().DynamicPlaceholder()` will just be calling the default overload with the placeholder key.

## Update your presentation

The big problem now is going to be your presentation in Sitecore. Because the format of the placeholder key has changed, all the existing renderings will need thier placeholder keys updating too. In a small implementation, you might be able to manually do this, but that wil take a long time!

### Sitecore PowerShell Extensions to the rescue

That is a phrase that have used so many times! SPE is going to help a lot here. If you have never used [Sitecore Powershell Extensions](https://marketplace.sitecore.net/Modules/Sitecore_PowerShell_console.aspx?sc_lang=en) - go download it now!

{% asset_img doitnow.gif "Just do it now!" %}

With SPE we can recurse through the content tree and update the presentation details. So lets look at a script that we can use to do that:

```PowerShell
$startPath = "/sitecore/content"
$placeholderKey = "main_*"

Write-Progress -Activity "Upgrading dynamic placeholders that for the key [$($placeholderKey)]" `
    -Status "Getting content items"

Get-ChildItem -Path $startPath -Recurse | ForEach-Object {
    $item = $_;

    Write-Progress -Activity "Upgrading dynamic placeholders that for the key [$($placeholderKey)]" `
        -CurrentOperation "Updating [$($item.Name)]..."

    Get-Rendering -Item $_  -Placeholder $placeholderKey | Foreach-Object {
        # Double check that this is an old style dynamic placeholder - should have a lowercase guid at the end of the string
        $matches = [regex]::Matches($_.Placeholder,'([0-9a-f]{8}[-][0-9a-f]{4}[-][0-9a-f]{4}[-][0-9a-f]{4}[-][0-9a-f]{12})$')
        if ($matches.Success) {
            $renderingId = $matches.Groups[0].Value

            # Replace the old style guid with the new style guid and add the unique number
            # for this upgrade, that will always be 0
            $newPlaceholder = $_.Placeholder.Replace($renderingId, "{$($renderingId.ToUpper())}_0")
            $_.Placeholder = $newPlaceholder
            Set-Rendering -Item $item -Instance $_
        }
    }
}

Write-Progress -Completed -Activity "Done"
```

This is a very basic script. First, set your start item. The script will recurse through the content tree from there and find all renderings that are in a placeholder that match the pattern in `$placeholderKey` - use a wildcard to find all dynamic placeholders.

Then for each rendering, we double check that the key contains a guid in the format of the old implementation - this will be the last element of the string and a lowercase guid with no braces.

Next, we build the new format placeholder key and set that on the rendering.

This script will need to be run as an admin user in Sitecore and depending on your content tree, could take a while to run. Future options would be to make this cope with more than just a single placeholder key per pass. But its a good place to start. Run this script for every dynamic placeholder key you have in your solution. Once done, you should be upgraded.

## Next Steps

This is just one way that an upgrade could be done. If you don't want to use SPE (What is wrong with you?) - the same approach could be used with some custom C# code. Whichever way you do it, upgrading to the new Sitecore Dynamic Placeholders is a pretty simple task and you can then get the benefits of the new additions.

Of course - upgrading is just the first step. Once you have done this, you really should look at how you have used Dynamic Placeholders and see whether you should change that based on some of the new features not previously available.

-- Richard