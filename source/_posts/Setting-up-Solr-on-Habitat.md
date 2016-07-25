title: "Setting up Solr on Habitat"
date: 2016-06-28 23:28:00
tags:
category: Sitecore
---

The [Sitecore Habitat](https://github.com/Sitecore/Habitat) project has got some great features in. One of my favourites is how the [Gulp](http://gulpjs.com/) file is setup to make getting the project running on your local machine a breeze! It really does simplify the work a developer normally has to do when joining an existing project.

### Whats Gulp all about then?
For those that have never used Gulp before, Gulp is a [nodejs](https://nodejs.org/en/) based task runner. [Getting started](https://github.com/gulpjs/gulp/blob/master/docs/getting-started.md) with Gulp is pretty simple as you can see from that link. In Visual Studio 2015, there is some really nice support for task runners like Gulp, so now we don't even need to leave the development environment.

In the Habitat project, Gulp is used for everything from setting up your Sitecore dependencies folder, transforming config files to publishing all the projects in the solution.

### How does SOLR fit in?
There have been a number of blog posts already on how to set Solr up with a Sitecore 8 installation: 

* [Configuring Solr search with Sitecore 8](http://goblinrockets.com/2015/01/30/configuring-solr-search-sitecore-8/)
* [Configuring Solr for use with Sitecore 8](https://sitecore-community.github.io/docs/search/solr/Configuring-Solr-for-use-with-Sitecore-8/)

etc...

One of the more annoying features is that you have to disable all the Lucene config files and enable all the Solr ones. At the last count this was nearly 20 files to enable, and another 20 to disable! In multiple folders too. 

There are a few solutions out already that use scripts to set this up. But I wanted to be able to integrate this into the Habitat Gulp script so it becomes a one stop shop.

### Gulp-Rename
The first option I looked at was `gulp-rename`. But because of the nature of task runners, doing the rename via a gulp pipe doesn't work very well. A gulp pipe works in memory, nothing is committed to disk until the final `.pipe(gulp.dest("path"));` call. So because the files are renamed in memory and then written to disk, it actually creates a copy of the files with the new name and leaves the existing ones alone.... Not helpful. This would require a separate task to then delete all the unwanted files, and that was also problematic.

### Exec and Powershell to the rescue!
To setup a powershell script is pretty simple to do for this, and it turns out that executing a powershell script from Gulp is easy too.

The powershell script I'm using is a modified version of a gist by [Patrick Perron](https://gist.github.com/patrickperrone) that switches between [Lucene and Solr](https://gist.github.com/patrickperrone/59b8745ee8b8ff9045b5).

The modified version, only switches from Lucene to Solr and expects the `$rootPath` to be passed into the function, rather than requesting input from the user.

```
function Set-SCSearchProvider($rootPath)
{
    $validInput = $true;
    #test that path is valid
    If (!(Test-Path -Path $rootPath))
    {
        Write-Host "The supplied path was invalid or inaccessible." -ForegroundColor Red;
        $validInput = $false;
    }

    If ($validInput)
    {
        Write-Host "Set to Solr." -ForegroundColor Yellow;
        $selectedProvider = "Solr";
        $deselectedProvider = "Lucene";

        #enumerate all config files to be enabled
        $filter = "*" + $selectedProvider + "*.config*";
        $filesToEnable = Get-ChildItem -Recurse -File -Path $rootPath -Filter $filter;
        foreach ($file in $filesToEnable)
        {
            Write-Host $file.Name;
            if (($file.Extension -ne ".config"))
            {
                $newFileName = [io.path]::GetFileNameWithoutExtension($file.FullName);
                $newFile = Rename-Item -Path $file.FullName -NewName $newFileName -PassThru;
                Write-Host "-> " $newFile.Name -ForegroundColor Green;
            }
        }

        #enumerate all config files to be disabled
        $filter = "*" + $deselectedProvider + "*.config*";
        $filesToDisable = Get-ChildItem -Recurse -File -Path $rootPath -Filter $filter;
        foreach ($file in $filesToDisable)
        {
            Write-Host $file.Name;
            if ($file.Extension -eq ".config")
            {
                $newFileName = $file.Name + ".disabled";
                $newFile = Rename-Item -Path $file.FullName -NewName $newFileName -PassThru;
                Write-Host "-> " $newFile.Name -ForegroundColor Green;
            }
        }
    }
}
```

This is saved to the root of the solution to a file called `setup-solr.ps1`

Now we can use a gulp task to execute that script:

```javascript
gulp.task("Setup-Solr-Config", function(callback) {
	exec("Powershell.exe -executionpolicy remotesigned . .\\setup-solr.ps1; Set-SCSearchProvider -rootPath '" + config.websiteRoot + "'",
		function(err, stdout, stderr) {
			console.log(stdout);
			callback(err);
	});
});
```

To enable exec you need to require it like this:
```javascript
var exec = require("child_process").exec;
```

### Breaking down the task
The gulp task is pretty simple. We execute `Powershell.exe` and pass in the required parameters. Make sure the execution policy is set so that we can run our script and then give it the file with the function to import. Then we can just call the function and pass in the parameter for the `$rootPath`. With Habitat, this is stored in the `gulp-config.js` file, which creates a `config` object containing all required settings.

`Exec` takes in 2 params, the command line to execute and a callback function to pass the output too. The callback function in this case, just logs the output from `stdout` to the console. Any errors are passed to the gulp task callback function.

Now the task appears in the task runner explorer and running it quickly renames all the required files to setup Habitat for Solr.

{% asset_img TaskRunnerExplorer.png "Task Runner Explorer" %}

### Expanding the task
We could make this even easier for developers. Some options to expand this task would include:

* Installing the binary files to the website instance from the Solr Support packge
* Setting up a config patch file if you wanted to rename the Cores for developer machines that have multiple projects on the same Solr instance
* Updating the Global.asax file to setup the IoC container with the right option for your project

-- Richard
