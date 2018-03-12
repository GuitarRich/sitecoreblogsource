---
title: Sitecore 9 Installation - Could not load file or assembly 'ChilkatDotNet46'
tags:
  - Installation
  - Sitecore
  - Sitecore 9
  - Errors
category: Sitecore
date: 2018-03-12 09:25:32
---

A collegue of mine was onboarding to a Sitecore 9 Update 1 project this week and had this error:

> Could not load file or assembly 'ChilkatDotNet46' or one of its dependencies. An attempt was made to load a program with an incorrect format.

Ahh - this is an easy one, I thought, he just didn't read the pre-requisites properly and forgot to install the Visual C++ 2015 Redistributable. I'm sure you will have read at this point that v9.0 update 1 introduced a dependency on this as it is needed by `ChilkatDotNet46.dll`.

So I helpfully pointed out that he should make sure all the pre-requistes should be installed!

{% asset_img install1.jpg "Yeah... that would be great" %}

But he had already installed the Visual C++ 2015 Redistributable on his machine. After some debugging and seaching around for the same error, the issue looked like it was a mismatch between 32 and 64 bit versions of the file.

Sitecore is requiring the 64 bit version of `ChilkatDotNet46`, but for some reason, ASP.NET was finding the 32 bit version. This post from Alex Brown ([Could not load file or assembly ‘ChilkatDotNet2’ or one of its dependencies. An attempt was made to load a program with an incorrect format.](http://www.alexjamesbrown.com/blog/development/could-not-load-file-or-assembly-chilkatdotnet2-or-one-of-its-dependencies-an-attempt-was-made-to-load-a-program-with-an-incorrect-format/)) pointed us in the right direction, although he had the reverse issue.

It turned out that the application pool that was created for the Sitecore 9 instance, had `Enable 32-Bit Applications` turned on. But default, this is disabled. But for some reason (which we have not yet worked out), on 2 of my collegues machines, it was getting set to true for new application pools:

{% asset_img apppoolsettings.png "Setting the Enable 32-Bit" %}

Once we disabled that setting, Sitecore came straight back up and we had no issues. So if you get that error, check your application pool settings and make sure that `Enable 32 Bit Applications` is disabled!

-- Richard.