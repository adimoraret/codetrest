---
layout:             post
title:              "Apply Dependency Inversion Principle to JS to avoid mocking libraries"
menutitle:          "Apply Dependency Inversion Principle to JS to avoid mocking libraries"
date:               2019-10-07 07:21:00
category:           JavaScript
author:             adimoraret
language:           EN
comments:           false
published:          true
---
## What is the problem with mocking libraries 
I've seen a lot of problems with mocking libraries: 
1. They come with their own set of rules, 
1. They have their own syntax
1. Code is hard to read/review for someone who is not familiar with the framework. 

This happens in all languages that I've worked with. However, in languages that support interfaces, it's easy to use. If your production code depends on an interface, rather than a concrete implementation, then in your test you can create a test-class that inherits from the interface and inject the test-class instead of the real-class. This is Dependency Inversion principle. That's because according to this principle High Level modules should not depend on Low Level modules. Implementations are details (low level) and they should depend on abstractions (high level). Interfaces are abstractions and they should not depend on details. 
But what can we do in JavaScript since there are no interfaces?

## How to use the Dependency Inversion Principle to avoid mocking libraries in JavaScript ##
JavaScript might not have the "interface" keyword at this moment, but it doesn't mean that we can't apply SOLID principles like "Interface Segregation" or "Dependency Inversion". We need to look at what does an interface mean, what is the goal of that interface and see if we find a way to do the same thing in JavaScript.
According to wikipedia, "In computing, an interface is a shared boundary across which two or more separate components of a computer system exchange information". Basically, it is a contract between components. Let's say we have a function that displays something. Typically, we would do something like this:

```javascript
async function getCurrentTemperature(zipCode){
    var thirdPartyApi = new ThirdPartyApi("some-api-key", "some-api-secret");
    const temperature = await thirdPartyApi.getCurrentTemperature(zipCode);
    console.log(`Result: ${JSON.stringyfy(temperature)}`);
    return temperature.now;
}
```
As we can see, this small function depends on two components:
1. A Third Party Api that provides temperature for a given zip code
2. A logging component that logs the api response

Writing a unit test for this is not straight forward because, when running this function, it will try to access the third party Api, and we'll need to provide some credentials as part of the test setup. There are some libraries which can mock the http response for given urls. Nock, is one of them. But this works only in HTTP world. What if we depend on a module that does I/O operations like file read/write?
How can we abstract these dependencies? One way is to have something like this:
```javascript
async function getCurrentTemperature(zipCode, temperatureApi, log){
    const temperature = await temperatureApi.getCurrentTemperature(zipCode);
    log(`Result: ${JSON.stringyfy(temperature)}`);
    return temperature.now;
}
```
In production code we could call it like this:
```javascript
const thirdPartyApi = new ThirdPartyApi(process.env.thirdPartyApiKey, process.env.thirdPartyApiSecret);
const currentTemperature = await getCurrentTemperature(zipCode, thirdPartyApi, console.log)
```
And in our unit test we would test the function like this:
```javascript
const mockApi = { getCurrentTemperature: (zipCode) => ( now: { fahrenheit: 86, celsius: 30 }) }; 
const noLog = (message) => {};
await getCurrentTemperature(zipCode, mockApi, noLog)
```
This would work and it would help avoiding using mocking tools, but I would go one step further and have an IoC container that would centralize all these components and use this IoC all over the application to pull these components (real in production code or mocks in unit tests)

```javascript
let thirdPartyApi = null;
let log = null;

class IoC {
    static set thirdPartyApi(value) {
        thirdPartyApi = value;
    }

    static get thirdPartyApi() {
        return thirdPartyApi;
    }

    static set log(value) {
        log = value;
    }

    static getLog() {
        return log;
    }
}
```
Production code becomes
```javascript
async function getCurrentTemperature(zipCode, temperatureApi, log){
    const temperature = await IoC.thirdPartyApi.getCurrentTemperature(zipCode);
    IoC.log(`Result: ${JSON.stringyfy(temperature)}`);
    return temperature.now;
}
```

When application starts, we need to inject real implementations:
```javascript
IoC.thirdPartyApi = new ThirdPartyApi(process.env.thirdPartyApiKey, process.env.thirdPartyApiSecret);
IoC.log = console.log;
```

When testing, we need to inject mocked implementations
```javascript
  it('should return temperature', async () => {
    const fahrenheit = 86;
    const celsius = 30;
    IoC.log = () => { };
    IoC.thirdPartyApi = { getCurrentTemperature: () => ({ now: { fahrenheit, celsius } }) };

    const temperature = await getCurrentTemperature('90210');

    expect(temperature.fahrenheit).to.equal(fahrenheit);
    expect(temperature.celsius).to.equal(celsius);
  });

  //if log is a required behaviour, we can easly test that it gets called like this
  it('should log when returning temperature', async () => {
    const fahrenheit = 86;
    const celsius = 30;
    let isLogCalled = false;
    IoC.log = () => { isLogCalled = true; };
    IoC.thirdPartyApi = { getCurrentTemperature: () => ({ now: { fahrenheit, celsius } }) };

    await getCurrentTemperature('90210');
    expect(isLogCalled).to.equal(true);
  });  
```

## Example ##
Full example can be found on [GitHub](https://github.com/adimoraret/ioc)
