---
title: Right Click Unlock Item with Sitecore PowerShell Extensions
tags:
  - Sitecore
  - Sitecore PowerShell Extensions
  - PowerShell
  - Users
category: PowerShell
date: 2016-11-01 20:52:47
---

I have been asked many times if it is possible for non admin users in Sitecore to be able to unlock items that have been locked by others. The short answer is no, the Sitecore command (`Sitecore.Shell.Framework.Commands.CheckIn`) only allows the user who owns the lock, or an administrator to unlock the item. And due to the fact that being an Adminstrator in Sitecore is _not_ a role, but a custom attribute on the user account, you can't assign this to any other user roles:

```cs
if (Context.IsAdministrator)
    return !obj.Locking.IsLocked() ? CommandState.Hidden : CommandState.Enabled;
```

But this is a feature that many clients want/need, its a simple thing to write a custom command to unlock the items, then you can code in a role for that command and assign users to that. Unlocking an item is as simple as:

```cs
item.Editing.BeginEdit();
item.Locking.Unlock();
item.Editing.EndEdit();
```

Not rocket science. But having to write that in code, compile the binary and deploy that, just seems like a lot of effort. Also you have to either hard code a role, or add some configuration to specify which role gets the unlock etc... 

### Sitecore PowerShell Extensions
With [Sitecore PowerShell Extensions][1] this task becomes trivial. So here is the first of my **Sitecore Nuts and Bolts** PowerShell script library scripts. I'll update this over the next few months with some scripts that I find really useful in my development and also for content editors too.

The SPE version of the unlocking code is even more simple than the C# version:

```ps
$item = Get-Item .
$item.Locking.Unlock()
Close-Window
```

Just to go through it - `Get-Item .` gets the current context item, in this case I have created the module on the right-click Context menu, so it will be which ever item you have selected in the Content Editor. The next line is self explanitory, finally `Close-Window` closes the SPE script window so the user doesn't have to..... Simples!

To make the *eXperience* just right for the user, I added a rule to the `Show if rules are met or not defined` field, so that this option will only show if the current item is already locked:

{% asset_img rule.png "Only show the menu if the item is locked" %}

### In Use
Now if I select the `/sitecore/Content/home` item and click the `Edit` button to lock the item. On right-clicking I get the option to `Unlock Item`:

{% asset_img unlockitem.png "Unlock the item" %}

On clicking the menu option, the item is unlocked:

{% asset_img itemunlocked.png "Unlock the item" %}

Now when right-clicking the menu option is not visible because of the rule for display:

{% asset_img nomenuitem.png "No menu item" %}


### Security
Now we can add Security in the standard Sitecore way. Just select the script module item, and add security to it like you would to any other Sitecore item:

{% asset_img security.png "Add security to the script module" %}

### Download it here!
If you want to try it out, download a package with the PowerShell Script Module items: {% asset_link SitecoreNutsandBoltsUnlockUser.zip "Unlock User - Sitecore Nuts & Bolts SPE Module" %}

- Richard

  [1]: https://marketplace.sitecore.net/en/Modules/Sitecore_PowerShell_console.aspx
