---
id: 124
title: 'Autofac + NServiceBus + IStartable &#8211; making them play nicely'
date: 2018-11-10T01:00:00+10:00
author: eddiewould
layout: post
guid: http://feedingthe.code.blog/2018/11/10/autofac-nservicebus-istartable-making-them-play-nicely/
permalink: /2018/11/10/autofac-nservicebus-istartable-making-them-play-nicely/
restapi_import_id:
  - 5be77bc7d1246
categories:
  - Uncategorized
---

Recipe to integrate the `IStartable` behaviour of AutoFac with NServiceBus.

As part of a recent project I was installing <a href="https://particular.net/nservicebus" target="_blank" rel="noopener noreferrer">NServiceBus</a> as a replacement for <a href="http://nimbusapi.com/" target="_blank" rel="noopener noreferrer">Nimbus</a> in an existing legacy system. The system makes use of <a href="https://autofac.org/" target="_blank" rel="noopener noreferrer">Autofac</a> for DI and adds various <a href="https://autofaccn.readthedocs.io/en/latest/lifetime/startup.html" target="_blank" rel="noopener noreferrer">IStartable</a> services to the container (background services etc).


The goal - to continue using Autofac as the DI container for both the application and the NServiceBus message handlers, <b>as well as</b> allow the `IEndpointInstance` (bus) to be injected into / consumed by `IStartable` services in the container. Removing the existing usages of `IStartable` wasn't really a viable option due to the size of the codebase / time constraints.

Here's how I tackled it:

```csharp
public static class BusRegisterExtensions
{
    public static void RegisterBusEndpoint(this ContainerBuilder builder, string endpointName)
    {
        builder.Register(context =>;
            {
                var busConnectionStringSetting = context.Resolve();
                var endpointConfig = CreateEndpointConfiguration(endpointName, busConnectionStringSetting);
 
                return new StartableEndpointInstanceProxy(endpointConfig, context.Resolve<ILifetimeScope>());
            })
            .As<IStartable>()
            .As<IEndpointInstance>()
            .SingleInstance();
    }
 
    public static EndpointConfiguration CreateEndpointConfiguration(string endpointName, BusConnectionStringSetting busConnectionString)
    {
        // ...
    }
}
 
internal class StartableEndpointInstanceProxy : IEndpointInstance, IStartable
{
    private readonly EndpointConfiguration _endpointConfig;
    private readonly ILifetimeScope _scope;
    private IEndpointInstance _instance;
 
    public StartableEndpointInstanceProxy(EndpointConfiguration endpointConfig, ILifetimeScope scope)
    {
        _endpointConfig = endpointConfig;
        _scope = scope;
    }
 
    public void Start()
    {
        _endpointConfig.UseContainer(customizations =>
        {
            customizations.ExistingLifetimeScope(_scope);
        });
 
        _instance = Endpoint.Start(_endpointConfig).ConfigureAwait(false).GetAwaiter().GetResult();
    }
  
    public async Task Send(object message, SendOptions options)
    {
        await _instance.Send(message, options);
    }
 
    public async Task Send(Action messageConstructor, SendOptions options)
    {
        await _instance.Send(messageConstructor, options);
    }
 
    // .. snip (omitted for brevity)
}
```

The idea behind the `StartableEndpointInstanceProxy` is to have the bus automatically started when the container is built (implements `IStartable`) whilst also passing the lifetime scope to the bus (*so the bus message handlers resolve components from the Autofac container*).

It creates the NServiceBus endpoint and then delegates the implementation of `IEndpointInstance` to it (implemenntation omitted for brevity).

This allows _other_ `IStartable` services to depend on the bus (IEndpointInstance) - Autofac specifies that IStartable dependencies will be started prior to any IStartable dependants.

We can register a type such as StartupSender to the container and everything "Just Works" (TM)

```csharp
public class StartupSender : IStartable
{
    private readonly IEndpointInstance _bus;
 
    public StartupSender(IEndpointInstance bus)
    {
        _bus = bus;
    }
 
    public void Start()
    {
        Task.Run(() =>; Send());
    }
 
    private async Task Send()
    {
        var evt = new StartupEvent()
        {
            Sent = DateTimeOffset.Now,
            Instance = Environment.MachineName,
        };
 
        await _bus.Publish(evt);
    }
}
```

`StartupSender` declares a dependency on `IEndpointInstance` (i.e. the bus). Because our registered implementation of `IEndpointInstance` is an `IStartable` and `StartupSender` depends on `StartableEndpointInstanceProxy`, the code in `StartableEndpointInstanceProxy::Start` (which creates the actual endpoint and configures it to use the lifetime scope from the Autofac container) will run _before_ the `StartupSender::Start` code - publishing to the bus will therefore succeed.