---
title: 'Sitecore Docker: Making Those Urls Pretty'
tags:
  - Sitecore
  - Docker
category: Docker
date: 2020-02-19 09:13:24
---


### Containers, Containers, Containers!

{% asset_img containerseverywhere.jpg %}

Getting Sitecore running on containers using Docker seems to be the current craze/buzzword. It's really gaining traction, but unlike a lot of fads, frameworks and buzzwords, this one actually is useful!

This blog post is not going to give you any tutorials or how-to's on setting up Sitecore on Docker. Some people far cleverer than I have already done that. If you want a simple no frills intro, you can't get much better than [Mark Cassidy's](https://intothecloud.blog/) series:

- [Docker for Dummies - Docker 101  or Docker Basics](https://intothecloud.blog/2019/09/14/Sitecore-Docker-for-Dummies/)
- [Docker for Dummies - Setting up Sitecore Docker images](https://intothecloud.blog/2019/09/16/Setting-up-Sitecore-Docker-Images/)
- [Docker for Dummies - Deploying & Debugging Visual Studio Solutions](https://intothecloud.blog/2019/09/21/Deploying-and-Debugging-Your-Visual-Studio-Solution-to-Your-Sitecore-Docker-Containers/)

### So What's with the Urls Then?

Getting Sitecore running on Docker, turns out to be pretty simple. Follow the steps in the articles above and soon you'll be loading up `http://localhost:44001` or `http://localhost:44002` in your browser. From this point, I'll assume you have managed to get Sitecore running in Docker on your local machine.

But here is where my problems started. First, if you are developing a multisite solution, `localhost` isn't going to cut it for long. As soon as you need a second site, you need domain names. Also, if you are working on multple client projects, `localhost` can get hard to keep track. Sure, you might only be running containers for one client at a time, but in the CM, it would be nice to have a quick glance at the url to remind yourself which project/client you are on. You might run multiple containers at once. Finally, I just don't like having to put the port number in the url. Yes, its just asthetics, and yes it does make me one of "those" people! But it is what it is, the port number is ugly - I like having pretty urls!

One more thing that I like to have for local development, is my sites running under SSL. That might seem overkill, and I agree 90% of the time it is. But I've seen too many bugs make it into the codebase because everything was tested on HTTP locally and then broke as soon as the protocol changed. Both via the `LinkManager` screwing things up, and by lazy developers doing silly things like hard coding the protocol! (insert your fav Picard face palm gif here!), and yes I agree that these are things that _shouldn't_ happen, we have all seen stupid issues like that. So I like to be in a position to test those kind of things before pushing the code out to QA.

### THIDI: Fixing the Url's

There are a few different approaches to "fixing" the urls for Docker to look pretty. [Michael West]() has a [Docker Repo](https://github.com/michaellwest/docker-https) that solves this using a `startup.ps1` PowerShell script that will modify the hosts file in the container, using a nifty little script by [Rob Ahnemann](https://github.com/RAhnemann/windows-hosts-writer).

My approach was slightly different. I ended up using IIS as a reverse proxy to the Docker containers. I had played around with other reverse proxies like [Traefik](https://containo.us/traefik/). But to have those monitoring port 80, it meant that IIS had to be turned off. As not all of our projects are containerized yet, this is not an option.

But getting IIS to act as a reverse proxy is pretty simple.

#### Prerequisites

First, we need to make sure that we have the following modules installed. These can be installed by the Web Platform Installer:

- Url Rewrite v2.1 or higher
- Application Request Routing v3 or higher

Once installed, you get a new icon in IIS when you click on the server. **Application Request Routing Cache**. First thing to do is fire up IIS, open up the `Application Requet Routing Cache` and then click `Server Proxy Settings`:

{% asset_img serverproxysettings.png %}

Now tick the checkbox to enable reverse proxies:

{% asset_img enablerp.png %}

#### Docker Compose Modifications

Now we have the basic setup in IIS. Its time to setup the `docker-compose.yml` file. I'll just show the relavent parts here:

At the bottom of the file, make sure you have your networks configured:

```yaml
networks:
  default:
    external:
      name: nat
```

Now for the CD and CM, you can add aliases:

```yaml
  cm:
    image: ${REGISTRY}sitecore-xp-sxa-standalone:${SITECORE_VERSION}-windowsservercore-${WINDOWSSERVERCORE_VERSION}
    entrypoint: powershell.exe -Command "& C:\\tools\\entrypoints\\iis\\Development.ps1"
    volumes:
      - .\startup:C:\startup
      - ..\build\Debug:C:\src
      - ..\src:C:\inetpub\wwwroot\app_data\unicorn
    ports:
      - "44001:80"
    networks:
      default:
        aliases:
          - myproject-cm.dev.local
    environment:
      HOST_HEADER: myproject-cm.dev.local
      SITECORE_LICENSE: ${SITECORE_LICENSE}
      SITECORE_APPSETTINGS_ROLE:DEFINE: Standalone
      SITECORE_APPSETTINGS_SXAXM:DEFINE: sxaxconnect
      SITECORE_CONNECTIONSTRINGS_CORE: Data Source=sql;Initial Catalog=Sitecore.Core;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_SECURITY: Data Source=sql;Initial Catalog=Sitecore.Core;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_MASTER: Data Source=sql;Initial Catalog=Sitecore.Master;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_WEB: Data Source=sql;Initial Catalog=Sitecore.Web;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_EXPERIENCEFORMS: Data Source=sql;Initial Catalog=Sitecore.ExperienceForms;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_SOLR.SEARCH: http://solr:8983/solr
      SITECORE_CONNECTIONSTRINGS_MESSAGING: Data Source=sql;Database=Sitecore.Messaging;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_XDB.MARKETINGAUTOMATION: Data Source=sql;Database=Sitecore.MarketingAutomation;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_XDB.PROCESSING.POOLS: Data Source=sql;Database=Sitecore.Processing.Pools;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_XDB.REFERENCEDATA: Data Source=sql;Database=Sitecore.ReferenceData;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_XDB.PROCESSING.TASKS: Data Source=sql;Database=Sitecore.Processing.Tasks;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_EXM.MASTER: Data Source=sql;Database=Sitecore.EXM.Master;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_REPORTING: Data Source=sql;Database=Sitecore.Reporting;User ID=sa;Password=${SQL_SA_PASSWORD}
      SITECORE_CONNECTIONSTRINGS_SITECORE.REPORTING.CLIENT: http://xconnect
      SITECORE_CONNECTIONSTRINGS_XCONNECT.COLLECTION: http://xconnect
      SITECORE_CONNECTIONSTRINGS_XDB.MARKETINGAUTOMATION.OPERATIONS.CLIENT: http://xconnect
      SITECORE_CONNECTIONSTRINGS_XDB.MARKETINGAUTOMATION.REPORTING.CLIENT: http://xconnect
      SITECORE_CONNECTIONSTRINGS_XDB.REFERENCEDATA.CLIENT: http://xconnect
      SITECORE_APPSETTINGS_TELERIK.ASYNCUPLOAD.CONFIGURATIONENCRYPTIONKEY: ${TELERIK_ENCRYPTION_KEY}
      SITECORE_APPSETTINGS_TELERIK.UPLOAD.CONFIGURATIONHASHKEY: ${TELERIK_ENCRYPTION_KEY}
      SITECORE_APPSETTINGS_TELERIK.WEB.UI.DIALOGPARAMETERSENCRYPTIONKEY: ${TELERIK_ENCRYPTION_KEY}
    depends_on:
      - sql
      - solr
      - xconnect
```

#### Configure the Reverse Proxy

To setup the Reverse Proxy website. My repo has a folder in just called `proxy` with the following `web.config` file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <outboundRules>
                <rule name="ReverseProxyOutboundRule_CM" preCondition="ResponseIsHtml1">
                    <match filterByTags="A, Form, Img" pattern="^http(s)?://localhost:44001/(.*)" />
                    <action type="Rewrite" value="http{R:1}://myproject-cm.dev.local/{R:2}" />
                </rule>
                <rule name="ReverseProxyOutboundRule_CD" preCondition="ResponseIsHtml1">
                    <match filterByTags="A, Form, Img" pattern="^http(s)?://localhost:44002/(.*)" />
                    <action type="Rewrite" value="http{R:1}://{ORIGINAL_HOST}/{R:2}" />
                </rule>
                <preConditions>
                    <preCondition name="ResponseIsHtml1">
                        <add input="{RESPONSE_CONTENT_TYPE}" pattern="^text/html" />
                    </preCondition>
                </preConditions>
            </outboundRules>
            <rules>
                <rule name="ReverseProxyInboundRule_CM" stopProcessing="true">
                    <match url="(.*)" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{HTTP_HOST}" pattern="^(myproject-cm\.dev\.local)$" />
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="http://localhost:44001/{R:1}" />
                </rule>
                <rule name="ReverseProxyInboundRule_CD" stopProcessing="true">
                    <match url="(.*)" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{HTTP_HOST}" pattern="^((site1|site2|site3)\.dev\.local)$" />
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="http://{HTTP_HOST}:44002/{R:1}" logRewrittenUrl="false" />
                    <serverVariables>
                        <set name="ORIGINAL_HOST" value="{HTTP_HOST}" />
                    </serverVariables>
                </rule>
            </rules>
        </rewrite>
        <urlCompression doStaticCompression="false" doDynamicCompression="false" />
    </system.webServer>
</configuration>
```

To create the IIS Website, a developer simply needs to run a couple of PowerShell scripts:

First, `Create-WildcardCert.ps1`, this just creates a `*.dev.local` cert for local development. There are various versions of this script floating about. I can't remember exactly the origins of this one, if its yours DM me on Slack and I'll add a credit here!

```PowerShell
Param(
    $WildCardDomain = "*.dev.local",
    $RootDomain = "dev.local",
    $WildCardCertName = "DO NOT TRUST Sitecore Local Development dev.local"
)
process {
    # Generate SSL cert
    $existingCert = Get-ChildItem Cert:\LocalMachine\Root | Where FriendlyName -eq $WildCardCertName
    if(!($existingCert))
    {
        Write-Host "Creating & trusting an new SSL Cert for $WildCardCertName"
 
        # Generate a cert
        # https://docs.microsoft.com/en-us/powershell/module/pkiclient/new-selfsignedcertificate?view=win10-ps
        $cert = New-SelfSignedCertificate -FriendlyName $WildCardCertName -Subject $RootDomain -DnsName $RootDomain,$WildCardDomain -CertStoreLocation "cert:\LocalMachine\My" -Type SSLServerAuthentication -NotAfter (Get-Date).AddYears(10)
        Export-Certificate -Cert $cert -FilePath "$PSScriptRoot\$RootDomain.cer"
        Import-Certificate -Filepath "$PSScriptRoot\$RootDomain.cer" -CertStoreLocation "cert:\LocalMachine\Root"
    }
}
```

Then we have `Create-ReverseProxy.ps1`. Pretty self explanitory, this script creates the AppPool and Web application in IIS:

``` PowerShell
try {
    Import-Module WebAdministration

    $iisAppPoolName = "myproject-reverse-proxy-app"
    $iisAppPoolDotNetVersion = "v4.0"
    $iisAppName = "myproject-reverse-proxy"
    $directoryPath = Get-Item $PSScriptRoot\..\proxy | % { $_.FullName }

    #navigate to the app pools root
    Set-Location -Path IIS:\AppPools\

    #check if the app pool exists
    if (!(Test-Path $iisAppPoolName -pathType container))
    {
        Write-Host "Creating the application pool"

        #create the app pool
        $appPool = New-Item $iisAppPoolName
        $appPool | Set-ItemProperty -Name "managedRuntimeVersion" -Value $iisAppPoolDotNetVersion
    }
    Write-Host "Trying to create the Site"

    #navigate to the sites root
    Set-Location -Path IIS:\Sites\

    #check if the site exists
    if (Test-Path $iisAppName -pathType container)
    {
        Write-Host "Site already exists"
        return
    }

    #create the site
    $iisApp = New-Item $iisAppName -bindings @{protocol="http";bindingInformation=":80:" + $iisAppName} -physicalPath $directoryPath
    $iisApp | Set-ItemProperty -Name "applicationPool" -Value $iisAppPoolName

    Write-Host "Adding Bindings"

    New-WebBinding -Name $iisAppName -IPAddress "*" -Port 443 -Protocol https -HostHeader "site1.dev.local"
    New-WebBinding -Name $iisAppName -IPAddress "*" -Port 443 -Protocol https -HostHeader "site2.dev.local"
    New-WebBinding -Name $iisAppName -IPAddress "*" -Port 443 -Protocol https -HostHeader "site3.dev.local"
    New-WebBinding -Name $iisAppName -IPAddress "*" -Port 443 -Protocol https -HostHeader "myproject-cm.dev.local"

}
finally {

    Write-Host "Done"
}
```

I'm not a PowerShell wizard, so I'm sure this script could be made better. Most of the time, adding those bindings will select the correct `*.dev.local` certificate. But it doesn't everytime and my PowerShell skills haven't been able to figure out why yet. I'm sure someone will have a better way of doing that.

### Running the Sites

Now that the reverse proxy is setup, all that remains is to run `docker-compose up`, wait for the conainers to start up and then hit one of those urls in your browser. You should see that Sitecore is responding nicely to the custom host names. There is no need to add a port number to the host and the Site Resolving should also work nicely.

As I mentioned above, there are a few ways of doing this. This is just one way that I have found that works well for our development team. I think Sitecore on Docker is a fantastic thing, it allows us to onboard new developers in minutes vs hours, and with just a few simple PowerShell scripts instead of complex and time consuming Sitecore installs. 

Thanks to everyone that wrote blog posts on Docker and who helps out a lot on the [Sitecore Slack](https://sitecore.chat) channel for `#docker`, big thanks to Michael West and Mark Cassidy for blogs, git repos and answering stupid questions when I had them :)

Do you have another way, a better way to do this? Let me know in the comments or hit me up on Sitecore Slack - I'm `@guitarrich`

Thanks

-- Richard.