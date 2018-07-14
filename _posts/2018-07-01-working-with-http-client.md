---
layout:             post
title:              "Working with HttpClient"
menutitle:          "Working with HttpClient"
date:               2017-07-01 07:23:00
category:           CSharp
author:             Adrian Moraret
language:           EN
comments:           false
published:          true
todo: Rename title, find 2 minutest time_wait settings;
---
(in progress...)
HttpClient looked very easy to use. Luckily before using it, I did a little research. Here are the most important findings(in my opinion) about using HttpClient:
* use HttpClient as a singleton object
* don't use Default properties like timeout or headers unless you're 100% sure the same settings will be used for all your HttpClient calls
* use HttpClient async methods
* don't mix async methods with. Result on other async methods. This could cause some deadlocks

## Problem and source of problem ##
HttpClient inherits IDisposable interface. This causes every csharp developer to think at this pattern:

```csharp
using(var httpClient = new HttpClient()) 
{
    var result = await client.GetAsync("http://localhost");
    Console.WriteLine(result.StatusCode);
}
```
This would work, but the problem with this is that even after application ends and connections will remain in a TIME_WAIT state for a while (depending on your setting TODO). 
Here is an example with 10 http parallel requests.
```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var iterations = new int[10];
        var httpResponses = new ConcurrentBag<HttpStatusCode>();
        var tasks = iterations.Select(async item =>
        {
            using (var client = new HttpClient())
            {
                var result = await client.GetAsync("http://localhost/");
                httpResponses.Add(result.StatusCode);
            }
        });
        await Task.WhenAll(tasks);
        foreach (var response in httpResponses)
        {
            Console.WriteLine(response);
        }
    }
}
```
If we run netstat.exe we'll see our current TCP/IP network connections. We'll run with the following parameters:
* "-a" to display all current connections
* "-o" to display the Process Id (PID) in order to identify it in Task Manager
* "-n" to display host addresses and ports in numerical form

```powershell
# Running in PowerShell
netstat -aon | select-string ":80"
```
Running the app and netstat.exe right after, should output something similar to:
```bash
C:\> dotnet .\HttpClientDemo.dll
OK 
OK 
OK 
OK 
OK 
OK 
OK 
OK 
OK 
OK 

C:\> netstat -aon | select-string ":80"

TCP    192.168.0.3:50855      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50856      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50857      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50858      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50859      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50860      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50861      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50862      192.30.252.154:80      TIME_WAIT       0
TCP    192.168.0.3:50863      192.30.252.154:80      TIME_WAIT       0
```

It looks like there are still Http connections in a TIME_WAIT state after our program finished. This state means that the connection is closed, but it's still waiting for  packets to come from destination. This might not be a problem in a non-prod environment, but if you have too many connections in this state, you might experience some errors.

## Use a single HttpClient object ##
Now, let's switch to a static HttpClient object.
```csharp
public class Program
{
    private static readonly HttpClient HttpClient = new HttpClient();

    public static async Task Main(string[] args)
    {
        var iterations = new int[10];
        var httpResponses = new ConcurrentBag<HttpStatusCode>();
        var tasks = iterations.Select(async item =>
        {
            var result = await HttpClient.GetAsync("http://localhost");
            httpResponses.Add(result.StatusCode);
        });
        await Task.WhenAll(tasks);
        foreach (var response in httpResponses)
        {
            Console.WriteLine(response);
        }
    }
}
```
Running again the app and netstat.exe right after and it would look like the problem is solved.

## Don't set global properties to static client ##

What if we want to change the timeout of the request? A typical complex app will request information from several resources which might have different expected timeouts. Let's look at the following code:
```csharp
public class Program
{
    private static readonly HttpClient HttpClient = new HttpClient();

    public static async Task Main(string[] args)
    {
        HttpClient.Timeout = TimeSpan.FromSeconds(10);
        var result = await HttpClient.GetAsync("http://localhost");
        Console.WriteLine(result.StatusCode);

        HttpClient.Timeout = TimeSpan.FromSeconds(20);
        result = await HttpClient.GetAsync("http://localhost");
        Console.WriteLine(result.StatusCode);
    }
}
 ```
The above code will generate an exception: `System.InvalidOperationException: This instance has already started one or more requests. Properties can only be modified before sending the first request.`

One way to fix this is to create a `HttpRequestMessage` object and pass this object to `HttpClient.SendAsync()` method while the timeout can be passed as a CancellationToken.
```csharp
public class Program
{
    private static readonly HttpClient HttpClient = new HttpClient();

    public static async Task Main(string[] args)
    {
        var iterations = new int[10];
        var httpResponses = new ConcurrentBag<HttpStatusCode>();
        var tasks = iterations.Select(async item =>
        {
            var timeout = TimeSpan.FromMilliseconds(300);
            var url = new Uri("http://codetrest.com");
            var cts = new CancellationTokenSource(timeout);
            using (var message = new HttpRequestMessage(HttpMethod.Get, url))
            {
                try
                {
                    var result = await HttpClient.SendAsync(message, cts.Token);
                    httpResponses.Add(result.StatusCode);
                }
                catch (Exception)
                {
                    httpResponses.Add(HttpStatusCode.RequestTimeout);
                }
            }
        });
        await Task.WhenAll(tasks);
        foreach (var response in httpResponses)
        {
            Console.WriteLine(response);
        }
    }
}
```
Given a small timeout could cause some intermediate timeouts like in the following example:
```bash
C:\> dotnet .\HttpClientDemo.dll
OK
OK
RequestTimeout
OK
OK
OK
RequestTimeout
OK
OK
RequestTimeout
```