title: "Multi-Item Publish with Sitecore Powershell Extensions"
date: 2015-12-14 15:53:16
tags:
category: Sitecore
---

A question came up on the Sitecore community today about being able to [publish multiple items in one go](https://community.sitecore.net/developers/f/8/p/2255/6614#6614). As a developer you can do this easily in [Sitecore Rocks](https://marketplace.sitecore.net/Modules/Sitecore_Rocks.aspx?sc_lang=en) as you can multi-select items in the tree and publish fromn the right-click menu. But in the Content Editor, there is no option to multi-select items. 

You can of-course publish all **sub-items** of an item, and you can publish **related items** also. An **Incremental** publish, runs through the current publish queue. But none of those options would allow the content editor to select a disparate set of items and publish them. It got me thinking, I wonder if I could do this with a [Sitecore Powershell Extensions](https://marketplace.sitecore.net/Modules/Sitecore_PowerShell_console.aspx?sc_lang=en) module/script.

If you haven't heard of it yet, Sitecore Powershell Extensions is, IMO, the best market place module you can get for Sitecore. I can't imagine how I used to get on without it being installed! [Watch this for a great overview of SPE](https://vimeo.com/134196432).

###SPE Module
At its core, an SPE module, is a group of scripts that work together to provide some functionality. In our case, we want to be able to:

* Create a new Publish Queue
* Add single items to the queue
* Add an item and all its children to the queue
* Publish/Process and clear the queue

###Session Persistency
A really nice feature of SPE is that you can have session persistency, that is variables set in one script can be read by another script. This allows us to have multiple steps.

{% asset_img sessionpersistency.png "Session Persistency" %}

###Muilti-Item Publish Module
Here are the scripts in our multi-item publish module:

{% asset_img modulescripts.png "Module Scripts" %}

So we start with a script that starts a new publish queue. For the queue, we will just use 2 arrays, 1 for the single items, and 1 for recursive publishes.

```powershell
# Initialize the arrays
$singleItems = @()
$branchItems = @()
Close-Window
```

Then we want to just add the currently selected item to each array:

```powershell
# Add Single Item 
$item = Get-Item .
$singleItems += $item
Close-Window
```

```powershell
# Add Branch Item 
$item = Get-Item .
$branchItems += $item
Close-Window
```

Finally - we want to process the items and publish them to all publish targets:

```powershell
$publishingTargetsFolderId = New-Object Sitecore.Data.ID "{D9E44555-02A6-407A-B4FC-96B9026CAADD}"
$targetDatabaseFieldId = New-Object Sitecore.Data.ID "{39ECFD90-55D2-49D8-B513-99D15573DE41}"

# Find the publishing targets item folder
$publishingTargetsFolder = [Sitecore.Context]::ContentDatabase.GetItem($publishingTargetsFolderId)
if ($publishingTargetsFolder -eq $null) {
    return $null
}

$totalItems = $singleItems.Length + $branchItems.Length

if ($totalItems -eq 0) {
    Write-Log "No items in the publish queue"
    return $null
}

# Retrieve the publishing targets database names
# Check for item existance in publishing targets
foreach($publishingTargetDatabase in $publishingTargetsFolder.GetChildren()) {
    Write-Log "Publishing items to the  $($publishingTargetDatabase[$targetDatabaseFieldId]) publishing target"
    
    $publishTarget = $publishingTargetDatabase[$targetDatabaseFieldId]
    $itemCount = 0
    
    $singleItems | Publish-Item -PublishMode Full -Target $publishTarget
    $branchItems | Publish-Item -Recurse -PublishMode Full -Target $publishTarget
}


Show-Alert "Finished publishing ${totalItems} items"

# Clear the publish queue
$singleItems = @()
$branchItems = @()

Close-Window
```

And hey presto, we have a module that allows the user to right-click items and queue them for publishing, and then just process that queue.

###Using the Module

Now a content editor can right-click an item, go to the `Publishing` menu under `Scripts`:

{% asset_img StartQueue.png %}

Then add items:

{% asset_img AddItems.png %}

Finally, publish and clear the queue:

{% asset_img PublishQueue.png %}

At the end, we give an indication of the items processed:

{% asset_img Finished.png %}

###This is just a start
This really was more for fun and to see if it was possible, so there are a number of features mssing that we might want to add. There is no language parameter, we might want to limit the publish targets etc... We could have better reporting on what is getting published too - but its a nice start.

The module code was losely based on the `Package Generator` module that ships with SPE.

{% asset_link MultiItemPublish-1.0.zip 'Click here to download a package to install the module!' %}

-- Richard Seal

{% asset_img SPEPoster.jpg %}

.