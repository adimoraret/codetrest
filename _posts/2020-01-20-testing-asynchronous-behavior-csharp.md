---
layout:             post
title:              "Testing asynchronous behavior in csharp"
menutitle:          "Testing asynchronous behavior in csharp"
date:               2019-10-07 07:21:00
category:           CSharp
author:             adimoraret
language:           EN
comments:           false
published:          false
---
## Why code with asynchronous behavior is hard to test ##
Whenever an asynchronous operation is performed, the result of the operation is not immediatly available to the caller. This is a very powerfull approach because it makes caller and callee decoupled. Caller can still run while callee performs operations(maybe long running operations) which would lead to a result. Because of this, web-pages or UI interfaces are fully responsive while asynchronious operations are being executed in background.

To get the result in an asynchronous operation, caller needs to:
* pass a callback to the callee and callee would make surre it calls the callback when result is available, or
* query periodically callee and ask if the result is available

### Passing a callback ###
This means that the callee needs to be designed to accept an input for the operation and a callback which will be executed when an operation is done. 
In Csharp it might look something like this:
```csharp
//TODO: Find a callback example
```
Moving from asynchronous low level to asynchronous system level, callbacks might be replaced with web hooks. Webhooks are endpoints that are exposed by the caller so the callee calls whenever result of the operation becomes available. In this case callbacks are sometimes passed as arguments next to the input, but sometimes are defined separately in an administration area.

Concrete example: Tracking emails. Caller sends an email to xyz@abc.com using an email provider service. Caller also exposes an API endpoint at /api/webhooks/open-email. Email provider service sends the email, and when users opens it, it will call the indicated API path.

### Query periodically callee for a response ###
This is when caller calls an asynchronous operation of the callee and callee quickly replies with a response that indicates it received the request. Usually there is a another operation provided by callee or some other client to pull the response. 
Algorithm could be this
```csharp
    var result = calle.GetResult();
    while (!result) 
    {
        if (breakLoopCondition) 
        {
            throw new NoResultException();
        }
        result = calle.GetResult();
        wait(); 
    }
```

## Testing ##
When testing asynchronous code, the actual test is the caller while the system under test is the callee                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           .

### Testing when asynchronous provides a callback ###

### Testing by quering periodically ###




## What about async/await ##