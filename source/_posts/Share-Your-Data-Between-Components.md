---
title: Share Your Data Between Components
tags:
  - Sitecore
  - Presentation
  - IoC
  - Components
category: Sitecore
date: 2017-11-02 19:35:45
---

Recently there was this blog post about [9 Grievances that make Sitecore development a challenge](https://blog.saberkarmous.nl/2017/11/nine-grievances-that-make-sitecore-a-challenge/), and while I agree at times, ~~Sitecore~~ development can be a challenge (although, isn't that really why we are developers - because of the challenge?), I wanted to address one of the grievances and how to solve that without too much effort.

## Yeah yeah, everything is an item. But where’s my page?

{% asset_img whereismypageobject.jpg "Where is my page?" %}

One of the greivances is that there is no concept of a "Page Object" that you can use to store something. For example, a dashboard that pulls in user information on 3 renderings, because of the way that Sitecore works, each controller action will get that data. So you end up getting the data 3 times. The common answer there would be to cache the data, but that isn't always the best option and depending on how your cache is invalidated, could lead to stale data being presented on the site.

## How to solve the problem?

The thing is that if you follow some good development practices, this perceived problem doesn't have to be a problem at all. Let's take the example of a dashboard with some user information that we are getting from a CRM. For the purposes here, we assume that the repository to call the CRM is written and called `ICRMRepository`.

The first thing to make sure is that we are following good practice and *not* doing everything in the controller. So we would have a service class that is responsible for getting user CRM data - lets call that `IUserCRMService`. The interface might look like this:

```csharp
public interface IUserCRMService
{
    UserData GetUserData(Guid userId);
}
```

What about the implementation, well this can have a dependency on the `ICRMRepository` and use that to get the data for the user. But instead of just using `IUserCRMService` as a stateless object, lets add some state to it. Lets add the user object to the service:

```csharp
public interface IUserCRMService
{
    UserData User { get; }
    void SetUserId(Guid Id);
}
```

Now we can implement that:

```csharp
public class UserCRMService : IUserCRMService
{
    private readonly ICRMRepository _crmRepository;

    public UserCRMService(ICRMRepository crmRepository)
    {
        _crmRepository = crmRepository;
    }

    public UserData User { get; private set;}

    public void SetUserId(Guid userId)
    {
        // Validate the user id

        // Don't get the data again if we already have it
        if (User != null && User.UserId == userId)
        {
            return;
        }

        var crmData = _crmRepository.GetUser(userId);
        User = new UserData
        {
            // add model mapping here
        };
    }
}
```

So now we can use the `SetUserId` to tell the service which user we are getting and then use the `User` property to get the data. Caveat - this is not supposed to be technically perfect and I'm sure there are neater ways to setup the stateful class, this is just to convey the idea!

The magic happens when we register the implementation with our container. For this I'll use Sitecore's builtin IoC services:

```csharp
public class Configurator : IServicesConfigurator
{
    public void Configure(IServiceCollection serviceCollection)
    {
        serviceCollection.AddScoped<IUserCRMService, UserCRMService>();
    }
}
```

Instead of the usual `Transient` registration, we are using `Scoped`. This means it is registered as a singleton per web request. So now we can inject that dependency (`IUserCRMService`) into as many controllers as we want and be sure that the CRM repository is only called once.

Ah - but what about race conditions? What if a second controller calls the service before the first one finishes? Well that _shouldn't_ happen in Sitecore, controllers are not called asynchronusly. But we _should_ is never a guarantee! So lets code against that.

{% asset_img racecondition.gif "I'm winning.... Oh no!" %}

Because we are a singleton per web request, we can safely use a lock here and not have to worry about requests blocking other requests coming in. So lets see the updated implementation of `IUserCRMService`:

```csharp
public class UserCRMService : IUserCRMService
{
    private object _userLock = new object();
    private readonly ICRMRepository _crmRepository;

    public UserCRMService(ICRMRepository crmRepository)
    {
        _crmRepository = crmRepository;
    }

    public UserData User { get; private set;}

    public void SetUserId(Guid userId)
    {
        // Validate the user id

        lock (_userLock)
        {
            // Don't get the data again if we already have it
            if (User != null && User.UserId == userId)
            {
                return;
            }

            var crmData = _crmRepository.GetUser(userId);
            User = new UserData
            {
                // add model mapping here
            };
        }
    }
}
```

Now if multiple controllers try to set the user Id at the same time, the first will lock and the others will wait until it is complete. The others will then see the `User` property populated and use that rather than calling the `ICRMRepository` again.

{% asset_img youarewelcome.jpg %}

I'm sure this is only one option of many to solve this problem, but its one that I have used a number of times and had great results with. Feel free to post comments on how you solve this below!

- Richard