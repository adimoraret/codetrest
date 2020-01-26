--- 
layout:             post 
title:              "Why Singleton Design Pattern doesn't violate SRP" 
menutitle:          "Why Singleton Design Pattern doesn't violate SRP" 
date:               2020-01-24 07:21:00 
category:           CSharp 
author:             adimoraret 
language:           EN 
comments:           false 
published:          false 
--- 

## Classical Singleton Design Pattern ## 
When talking about singleton design pattern, I'm talking about the classical design. Once frameworks which help with the inversion of control appeared, there is a simple way to create singletons. In the newer approach, objects get injected with different scopes and Singleton is one of them. 

Here is one simple, non-thread-safe way, to create a class in CSharp using classical singleton design pattern: 

```csharp 
public sealed class MySingleton 
{ 
    private static MySingleton _instance = null; 
    private MySingleton() { } 
    public static MySingleton Instance 
    { 
        get 
        { 
            if (_instance == null) 
            { 
                _instance = new MySingleton(); 
            } 
            return _instance; 
        } 
    } 
    public void OperationX() { } 
} 
``` 

The above example is simple enough. We design a class and we expect to have exactly one instance, no matter how many times we call ```MySingleton.Instance```. Obviously, this class has some behavior and we're exposing some operations (OperationX) as a method. 

## Single responsibility principle ## 
This principle states that every system, module, class, function should have a single responsibility, or as Robert C. Martin mentions, one and only one reason to change.  
Looking at the above example, we can see that it has at least two reasons to be changed: 
* one possible reason would be to change the algorithm that handles instance creation.  
* second possible reason would be any change related to the behavior of our object.  

Example: 
* Let's say that our system becomes widely used and because of race conditions we might end up with issues. Let's say two threads are checking in the same time if _instance is null, response is true for both and both will assign new instances to the same static variable. So, we'll need to change the instance creation algorithm. 
* Let's say OperationX is no longer enough and we need to add OperationY next. 

Code would look like this after these changes: 
```csharp 
public sealed class MySingleton 
{ 
    private static MySingleton _instance = null; 
    private static object _lock = new object(); 
    private MySingleton() { } 
    public static MySingleton Instance 
    { 
        get 
        { 
            if (_instance == null) 
            { 
                lock(_lock) 
                { 
                    if (_instance == null) 
                    { 
                        _instance = new MySingleton(); 
                    } 
                } 
            } 
            return _instance; 
        } 
    } 
    public void OperationX() { } 
    public void OperationY() { } 
} 
``` 

## Why these two changes are not SRP violation ? ## 
Even if both totally different reasons we had caused the same physical class to be changed, I would argue this is not SRP violation. The reason is because changes occur in different scopes: 
* change to the creation algorithm it's done to the *MySingleton Type*. Changed method and newly added property are static and they're connected to the type.  
* change to the behavior of the class would affect the *MySingleton Instance*. 

## But both changes are in the same class, in the same file ## 

Well, in CSharp it's easy to split this class into two files by using partial class. 
```csharp 
// Singleton.Type.cs 
public sealed partial class MySingleton 
{ 
    private static MySingleton _instance = null; 
    private static object _lock = new object(); 
    public static MySingleton Instance 
    { 
        get 
        { 
            if (_instance == null) 
            { 
                lock(_lock) 
                { 
                    if (_instance == null) 
                    { 
                        _instance = new MySingleton(); 
                    } 
                } 
            } 
            return _instance; 
        } 
    } 
} 

// Singleton.Instance.cs 
public sealed partial class MySingleton 
{ 
    private MySingleton() { } 
    public void OperationX() { } 
    public void OperationY() { } 
} 
``` 

I'm not a fan of partial classes, I try to avoid using them, but this is a way we can separate the behavior of the type vs the behavior of the instance although it's the same class. 