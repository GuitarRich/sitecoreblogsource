title: "Octopus Deploy Step for Unicorn Sync"
date: 2016-03-14 10:52:26
tags:
category: Sitecore
---

Two products that I love and have become an essential part of my Sitecore life are [Unicorn](https://github.com/kamsar/Unicorn) and [Octopus Deploy](https://octopus.com) - I plan on writing about my adventures with setting up CI/CD using Octopus Deploy soon, but for now here is a useful Step Template for calling the Unicorn sync methods remotely during your deployment in Octopus.

For the purposes of this post, I'll assume that you are familiar with both Octopus and Unicorn.

To install the step template, copy the json string at the bottom of the post and import into the Step Template library:

{% asset_img import-step.png "Import Step Template" %}

This will give you a new template you can use in your projects. Add the step to your project and you will get the following options:

{% asset_img step-template.png %}

Fill in the following fields:

* **Shared Secret** : This is the shared secret key that is held in the `Unicorn.UI.config` file. See this page for instructions on setting up the automated tool security: [https://github.com/kamsar/Unicorn#use-the-automated-tool-api](https://github.com/kamsar/Unicorn#use-the-automated-tool-api)
* **Site Url** : Set this to the url of the site that the Octopus Tentacle can use to call the unicorn api. You can type the url here or use an Octopus Variable
* **MicroCHAP DLL Location** : You need to make sure that the **MicroCHAP.dll** file is included in your project and that the Tentacle canread that file. This is for the newer style authentication that Unicorn uses
* **Configurations** : This is a list of all the configurations that you want to sync. You can add as many as you want here, separate each configuration with a new line

And thats it. Add that to your deployment process and it will automate the Unicorn sync.

###Whats in the step template?
For those that are interested in the PowerShell script, here it is:

```powershell
$ErrorActionPreference = 'Stop'

Add-Type -Path "${MicroChap}\MicroCHAP.dll"

Function Sync-Unicorn {
	Param(
		[Parameter(Mandatory=$True)]
		[string]$ControlPanelUrl,

		[Parameter(Mandatory=$True)]
		[string]$SharedSecret,

		[Parameter(Mandatory=$True)]
		[string[]]$Configurations,

		[string]$Verb = 'Sync'
	)

	# PARSE THE URL TO REQUEST
	$parsedConfigurations = ($Configurations) -join "^"
	$url = "{0}?verb={1}&configuration={2}" -f $ControlPanelUrl, $Verb, $parsedConfigurations

	Write-Host "Sync-Unicorn: Preparing authorization for $url"

	# GET AN AUTH CHALLENGE
	$challenge = Get-Challenge -ControlPanelUrl $ControlPanelUrl
	Write-Host "Sync-Unicorn: Received challenge: $challenge"

	# CREATE A SIGNATURE WITH THE SHARED SECRET AND CHALLENGE
	$signatureService = New-Object MicroCHAP.SignatureService -ArgumentList $SharedSecret
	$signature = $signatureService.CreateSignature($challenge, $url, $null)
	Write-Host "Sync-Unicorn: Created signature $signature, executing $Verb..."

	# USING THE SIGNATURE, EXECUTE UNICORN
	$result = Invoke-WebRequest -Uri $url -Headers @{ "X-MC-MAC" = $signature; "X-MC-Nonce" = $challenge } -TimeoutSec 10800 -UseBasicParsing
	$result.Content
}

Function Get-Challenge {
	Param(
		[Parameter(Mandatory=$True)]
		[string]$ControlPanelUrl
	)

	$url = "$($ControlPanelUrl)?verb=Challenge"
	$result = Invoke-WebRequest -Uri $url -TimeoutSec 360 -UseBasicParsing
	$result.Content
}

$configs = $Configurations.split("`n")
Sync-Unicorn -ControlPanelUrl "$($SiteUrl)/unicorn.aspx" -SharedSecret $SharedSecret -Configurations $configs
```

First we set the `$ErrorActionPreference`, this tells Octopus what the default action to take on an error is. This can be overridden in the process step when you add it to a project.

Next we add the `MicroCHAP.dll` so that it can be used by the rest of the script. The `Sync-Unicorn` and `Get-Challenge` functions are from the `Unicorn.psm1` file in the repo [https://github.com/kamsar/Unicorn/tree/master/doc/PowerShell%20Remote%20Scripting](https://github.com/kamsar/Unicorn/tree/master/doc/PowerShell%20Remote%20Scripting).

Finally we split the configurations variable on a new line and call the `Sync-Unicorn` function with all the variables set in the UI. Simples :)

##TL/DR;

Here is the json object that you can use to import the Step Template into Octopus Deploy:

{% gist 0d3b30ec36a2368b7b1e %}


Here is a gist that contains the step template for importing into Octopus - [OctopusDeploy-Unicorn-Step-Template.json](https://gist.github.com/GuitarRich/0d3b30ec36a2368b7b1e#file-octopusdeploy-unicorn-step-template-json)