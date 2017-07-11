title: "Sitecore 8 Bug Uploading Images in Zip Files"
date: 2015-09-15 17:10:02
tags:
- Sitecore
- Upload
- Sitecore Powershell Extensions
category: Sitecore
---

One of the nicer features in the Sitecore Media Library was the ability to upload a zip file of images and have it unpack them automatically. In the latest versions of Sitecore (version 8 update 4 and update 5 at the time of writing), this functionality is broken. The file uploads as a zip file and does not unpack.

### Steps to Reproduce:

* Load the Media Library
* Click **Upload Files (Advanced)**
* Select a zip file containing your images
  * Make sure you tick **Unpack Zip Archives**

{% asset_img upload-dialog.png "Sitecore Upload Dialog" %}

* Upload the file

The file is uploaded as a zip file and not unpacked.

{% asset_img zipflie.png "File Uploaded as Zip" %}

This has been reported to Sitecore support and acknowledged as a bug in the MediaFolder.js file.  If you are having the same issue, contact sitecore support and they will give you a fixed **MediaFolder.js** file. The public reference number for this bug report is 439231.

### Work Arounds
[Adam Najmanowicz](http://blog.najmanowicz.com/) helped out with a cool little SPE ([Sitecore Powershell Extentions](https://marketplace.sitecore.net/Modules/Sitecore_PowerShell_console.aspx?sc_lang=en)) script to upload the zip file and unpack. It appears that it is just the dialog in Sitecore 8 that is broken.

```
Receive-File -ParentItem (gi "master:\media library\your folder") -AdvancedDialog
```

---

{% asset_img spe-upload.png "SPE Upload Dialog" %}

Which worked perfectly! Thanks Adam!

If you haven't already tried it, I highly recommend installing SPE and trying it out! and if you like it, don't forget to rate it and give it a :thumbs up:

-- Richard