---
title: Getting to the FXM Beacon using TLS 1.2
tags:
  - Sitecore
  - FXM
  - Security
date: 2017-07-24 14:45:18
---

I have recently been playing around with FXM - I do like how simple it is to setup and get running. But I hit an issue when trying to connect FXM to a client site that had been secured using a certificate that used TLS 1.2 security protocal.

When FxM tried to search for the beacon JavaScript or when its trying to load in the pages for Experience Editor:

> 47456 14:38:10 ERROR The request was aborted: Could not create SSL/TLS secure channel. Exception: System.Net.WebException Message: The request was aborted: Could not create SSL/TLS secure channel. Source: System at System.Net.WebClient.DownloadDataInternal(Uri address, WebRequest& request) at System.Net.WebClient.DownloadString(Uri address) at Sitecore.FXM.Service.Abstractions.WebClientWrapper.Download(Uri address) at Sitecore.FXM.Service.Controllers.BeaconController.Ping(Uri address)

At first I thought there may be an issue with the cerificate, but that worked fine in browsers. So using the Sitecore developers favorite tool - dotPeek (or other decompiler) - I started tracing the code where the error was occuring.

## The Sea of Dissapointment

It turns out that FXM just uses a simple `WebClient` call to download the target html serverside and then it uses that to check for the beacon script and/or to load in the Experience Editor. Here is the code:

```csharp
namespace Sitecore.FXM.Service.Abstractions
{
    public class WebClientWrapper : IWebClient
    {
        private readonly WebClient client;

        public WebClientWrapper(WebClient client)
        {
            this.client = client;
        }

        public string Download(Uri address)
        {
        try
        {
            return this.client.DownloadString(address);
        }
        catch (WebException ex)
        {
            throw;
        }
        }
    }
}
```

The problem here is that by default, the `System.Net.WebClient` class uses `System.Net.ServicePointManager.SecurityProtocal` to control which security protocal version to use when creating the request object. By default this is set to `SecurityProtocolType.Tls|SecurityProtocolType.Ssl3` in .Net 4.0/4.5 - adding `SecurityProtocolType.Tls12` to this solves the issue - but where to add that.

I got excited at first, because this class is in the `Sitecore.FXM.Service.Abstractions` and implements an interface `IWebClient` - Bingo! I can just write a new implementation of that class, set the security protocal and register it instead of the standard Sitecore one. 

{% asset_img dissapointed.jpg "Nope!" %}

How silly of me! I guess the FXM team haven't got to fully implementing DI yet. Looking at the list of registrations in Sitecore (`/sitecore/admin/showserviceconfig.aspx`), the `IWebClient` and `WebClientWrapper` classes were not listed. Then looking at the `BeaconController` - which did have dependencies injected into the constructor, there was another constructor doing poor mans DI:

```csharp
   public BeaconController(IRepository<BeaconEntity> repository)
      : this(repository, (ILog) new LogWrapper(), (ICorePipeline) new CorePipelineWrapper(), (ITrackerProvider) new TrackerProviderWrapper(), (ISitecoreContext) new SitecoreContextWrapper(), (ISettings) new SettingsWrapper(), (IRequestHelper) new RequestHelper(), (HttpContextBase) new HttpContextWrapper(HttpContext.Current), (IWebClient) new WebClientWrapper(new WebClient()), (ITrackingManager) new TrackingManager((ICorePipeline) new CorePipelineWrapper(), (ITrackerProvider) new TrackerProviderWrapper(), (ISitecoreContext) new SitecoreContextWrapper()))
    {
    }
```

There doesn't seem to be a good place in any of the FXM code to set the security protocal.

## Good Old Initialize Pipeline!

Fortunately - the `System.Net.ServicePointManager` class is a static class (_never thought I'd hear myself saying that!_) - So if we set that protocal when the application initializes, as long as nothing else changes it back, we should be good. So thats what I did. Create a simple `initialize` pipeline processor and set the protocal:

```csharp
namespace Sitecore.Foundation.SitecoreExtensions.Pipelines.Initialize
{
    using System.Net;
    using Sitecore.Diagnostics;
    using Sitecore.Pipelines;

    public class SetupSecurityProtocol
    {
        public void Process(PipelineArgs args)
        {
            Log.Info("Setting SecurityProtocol to include TLS 1.2", this);

            ServicePointManager.SecurityProtocol = ServicePointManager.SecurityProtocol | SecurityProtocolType.Tls12;
        }
    }
}
```

Notice that, I'm keeping the original values too - this makes sure that it will still work with other versions of TLS or SSL.

Now patch that in:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/" xmlns:set="http://www.sitecore.net/xmlconfig/set/">
    <sitecore>
        <pipelines>
            <initialize>
                <processor type="Sitecore.Foundation.SitecoreExtensions.Pipelines.Initialize.SetupSecurityProtocol, Sitecore.Foundation.SitecoreExtensions"/>
            </initialize>
        </pipelines>
    </sitecore>
</configuration>
```

And finally - FXM can now create the connection correctly to a website certified with the TLS 1.2 protocal!

Hopefully this will help someone out there who has the same issue!