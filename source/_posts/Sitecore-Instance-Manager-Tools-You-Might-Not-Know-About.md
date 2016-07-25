title: "Sitecore Instance Manager - Tools You Might Not Know About"
date: 2015-08-11 14:10:26
tags:
- Sitecore
- SIM
- Developer Tools
category: Sitecore
---

I've been using [Sitecore Instance Manager](https://marketplace.sitecore.net/modules/sitecore_instance_manager.aspx) for a while now, but recently I noticed a few tools that for some reason I'd never seen before. So just incase you haven't seen them, here is a roundup of the cool things you can do with SIM.

Just look at all the cool things under the bundled tools menu! 
{% asset_img BundledToolsMenu.JPG "SIM Bundled Tools Menu" %}

### Hosts File Editor
No more running Notepad++ as an administrator and remember the path for the hosts file! Just make sure SIM is run with elevated privelleges and you can edit the hosts file right there. Simple! 

{% asset_img HostsEditor.JPG "SIM Hosts Editor" %}

### MS SQL Manager
Here you can attach and restore Sitecore databases. It even has a simple SQL Shell for running SQL Scripts against a database. Useful for something quick, or if you don't have access to SQL Server Management Studio for some reason! 

{% asset_img SQLServerManager.JPG "SIM MS SQL Manager" %}

### Other Tools

* **Config Builder** : You can look at a fully built config file by selecting the sites web.config file location. It merges all the include files and saves the resulting XML. Useful for debugging errors in the config that might stop the site running so you can't get to the **/sitecore/admin/showconfig.aspx** page.

* **Install MongoDB** : Nice and simple installer, saves you having to do it manually!

* **LinqPad, Log Analyzer, Standalone Rocks, SSPG** : Quick links to some useful tools here, especially the **Log Analyzer** a and the **Sitecore Support Package Generator**

### Site Tools
Of course there are also the tools you probably use more often that are specific to a sitecore instance:

#### The **Open** Ribbon
{% asset_img SIMSiteOpenMenu.JPG "SIM Site Context 'Open' Menu" %}
Here you have links to open the site, login, go to the Desktop, Content Editor or Experience Editor. Also you can open the site in a specified browser.

We can open the site folders, look at the many configuration files and even view the ShowConfig or fully merged web.config. There are also quick links to the log files and to the Visual Studio solution if it is in the instance folder. 

#### The **Edit** Ribbon
{% asset_img SIMSiteEditMenu.JPG "SIM Site Context 'Edit' Menu" %}
Here you can stop and start your site instance. Delete or Re-Install or the very useful *Multiple Delete* option. You can publish the site right from SIM and also Backup/Restore instances.

### Conclusion
Overall - SIM is a great tool for any Sitecore developer. I use it daily and I haven't even gone into the many options you have to extend/customize it and how you can use it via scripting.

Also the future is looking good for SIM - It will be going open source, so get your suggestions into the team [Sitecore Instance Manager Wiki](https://bitbucket.org/sitecore/sitecore-instance-manager/wiki/Home).


-- Richard