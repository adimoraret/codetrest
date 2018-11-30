---
layout:             post
title:              "Create service fabric stateless service with service bus"
menutitle:          "Create service fabric stateless service with service bus"
date:               2018-11-30 07:23:00
category:           Azure
author:             adimoraret
language:           EN
comments:           false
published:          false
---
## What I will implement ##
I'll create a Service Fabric stateless service in .Net Core 2.1. This service will be connected to an Azure Service Bus queue. I'll use Microsoft.Extensions.DependencyInjection for IoC and Microsoft.Extensions.Configuration to read application configuration file. 
Technologies and methods:
* Azure Service Fabric
* Azure Service Bus
* .Net Core 2.1
* IoC
* Environment configuration
* Visual Studio 2017

## Creating Visual Studio Project ##
Creating service fabric project
![create service fabric project](/assets/posts/2018-11-30/service-fabric-project.png)
Selecting stateless service
![select stateless service](/assets/posts/2018-11-30/service-fabric-stateless-service.png)

## Adding Dependency Injection and configuration ##
When the stateless service is generate it comes with the following code:
```csharp
  ServiceRuntime.RegisterServiceAsync("ServiceBusProcessorType",
      context => new ServiceBusProcessor(context)).GetAwaiter().GetResult();
```
I'll extract ```new ServiceBusProcessor(context))``` into a method so code becomes:
```csharp
  ServiceRuntime.RegisterServiceAsync("ServiceBusProcessorType",
      context => CreateServiceBusProcessor(context)).GetAwaiter().GetResult();
  //some code
  private static ServiceBusProcessor CreateServiceBusProcessor(StatelessServiceContext context)
  {
      return new ServiceBusProcessor(context);
  }
```
Now I'll install Microsoft.Extensions.DependencyInjection, Microsoft.Extensions.Configuration, Microsoft.Extensions.Configuration.Json, Microsoft.Azure.ServiceBus and Microsoft.Extensions.PlatformAbstractions into ServiceBusProcessor. 
![solution after nuget package installation](/assets/posts/2018-11-30/solution-after-nuget-package-installation.png)

Next I'll create a method to resolve current environment name. In case it's not defined it will default it to Development
```csharp
  private static string GetEnvironmentName()
  {
      var environment = Environment.GetEnvironmentVariable("SERVICE_FABRIC_ENVIRONMENT");
      return string.IsNullOrEmpty(environment) ? "Development" : environment;
  }
```
It's always a must to have environmental based configuration. I really like the new application configuration structure introduced by .NET Core. So I'll create 3 application configuration files and I'll make sure they're copied to output directory (right click -> properties):
* appsettings.Development.json
* appsettings.Stagig.json
* appsettings.Production.json
And here is how we can reference them from our service depending on the environment.
```csharp
  private static IConfigurationRoot GetConfiguration()
  {
      var environmentName = GetEnvironmentName();
      var basePath = PlatformServices.Default.Application.ApplicationBasePath;
      var configurationBuilder = new ConfigurationBuilder()
          .SetBasePath(basePath)
          .AddJsonFile($"appsettings.{environmentName}.json");

      return configurationBuilder.Build();
  }
```
Now, let's do IoC. I'm creating the following class.
```csharp
  internal static class Startup
  {
    public static void ConfigureIoc(ServiceCollection services, IConfiguration configuration, StatelessServiceContext context)
    {
        services.AddSingleton<ServiceContext>(context);
        services.AddSingleton<StatelessServiceContext>(context);
        services.AddTransient<IConfiguration>(s => configuration);
        services.AddTransient<ServiceBusProcessor>();
        services.AddTransient<IQueueClient>( s =>
        {
            var config = s.GetService<IConfiguration>();
            var messageBusSection = config.GetSection("AppSettings:MessageBus");
            var connectionString = messageBusSection["ConnectionString"];
            var queue = messageBusSection["Queue"];
            return new QueueClient(connectionString, queue);
        });
    }
  }
```
And inside of the main class we'll integrate it like this:
```csharp
  //and the very first method we created
  private static ServiceBusProcessor CreateServiceBusProcessor(StatelessServiceContext context)
  {
      return new ServiceBusProcessor(context);
  }
  //becomes
  private static ServiceBusProcessor CreateServiceBusProcessor(StatelessServiceContext context)
  {
      var services = new ServiceCollection();
      var configuration = GetConfiguration();
      Startup.ConfigureIoc(services, configuration, context);
      var serviceProvider = services.BuildServiceProvider();
      return serviceProvider.GetService<ServiceBusProcessor>();
  }
```

## Adding Service Bus ##
In ServiceBusProcessor.cs we'll take advantage of IoC and we'll include IQueueClient into its constructor. In the main method I'll connect to Service Bus and hook to two possible outcomes (success or error) to two private methods.
```csharp
  internal sealed class ServiceBusProcessor : StatelessService
  {
      private readonly IQueueClient _queueClient;

      public ServiceBusProcessor(StatelessServiceContext context, IQueueClient queueClient)
          : base(context)
      {
          _queueClient = queueClient;
      }

      protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
      {
          return new ServiceInstanceListener[0];
      }

      protected override async Task RunAsync(CancellationToken cancellationToken)
      {
          await Task.Run(() => Configure(OnMessageReceived, OnExceptionThrown), cancellationToken);
      }

      private void OnMessageReceived(string label, string message)
      {
          ServiceEventSource.Current.ServiceMessage(Context, $"Received message: {message} label: {label}");
      }

      private void OnExceptionThrown(Exception exception)
      {
          ServiceEventSource.Current.ServiceMessage(Context, $"Exception: {exception}");
      }

      private void Configure(Action<string, string> onMessageReceived, Action<Exception> onExceptionThrown)
      {
          var options = new MessageHandlerOptions(e =>
          {
              onExceptionThrown(e.Exception);
              return Task.CompletedTask;
          })
          {
              AutoComplete = false,
              MaxAutoRenewDuration = TimeSpan.FromMinutes(1),
          };
          _queueClient.RegisterMessageHandler(async (message, token) =>
          {
              var messageBody = Encoding.UTF8.GetString(message.Body);
              var label = message.Label;
              onMessageReceived(label, messageBody);
              await _queueClient.CompleteAsync(message.SystemProperties.LockToken);
          }, options);
      }
  }
```
Let's create service bus queue in Azure
![azure create service bus](/assets/posts/2018-11-30/azure-create-service-bus.png)
Let's Run Service Fabric project from Visual Studio and let's test it using Service Bus Explorer
![send message](/assets/posts/2018-11-30/send-message-to-service-bus.png)
Then it should show in Visual Studio -> Diagnostic Events window 
![message received in service bus](/assets/posts/2018-11-30/message-received-in-service-bus.png)

