--- 
layout:             post 
title:              "Order of interfaces matters" 
menutitle:          "Order of interfaces matters" 
date:               2020-02-28 07:21:00 
category:           CSharp
author:             adimoraret 
language:           EN 
comments:           false 
published:          true
--- 

## I didn't know either ## 
Here is something that I normally wouldn't think when I add a new interface as inheritable for a class that already has an interface.
Here is an example:
1. I have 


Whenever I want to test algorithms or formulas, I use London style testing. Here is a simple example: Let's say we want to calculate final price of a product for a given state tax.

## Chicago style ##
Most commonly, since according to this style we only care about the result, then we would implement it like this (for simplicity, I'm excluding here typical validation code for given arguments):
```csharp
public decimal GetFinalPrice(decimal price, decimal stateTax)
{
    return price + price * stateTax;
}
```
Here is how a Chicago style unit test would look like:
```csharp
[TestCase(100, 7.25, 107.25)]
public void ShouldCalculateFinalPrice(decimal price, decimal stateTax, decimal expectedFinalPrice)
{
    var finalPrice = _sut.GetFinalPrice(100, 7.25m);

    Assert.That(finalPrice, Is.EqualTo(expectedFinalPrice));
}
```
Now, this is correct, but let's say that by accident, we change the production code to:
```csharp
public decimal GetFinalPrice(decimal price, decimal stateTax)
{
    return price + stateTax;
}
```
Test would still pass. Isn't it?
Of course, we can write more tests then, many different inputs, and then we'll address most of the issues. But it won't be enough. How can we be sure that for every possible combination, our formula would work? We could add more tests, more values, but mathematically we can't prove a formula is correct just by looking at inputs and outputs. 

## London style ##
Since we care about behavior, then we should expect these behaviors:
 - a state tax is applied to the product's original price
 - result is determined by the sum of the original price and the result of the above operation
 - since the result might have more than two decimals, we'll need to round final result to 2 decimals.

So, we put the behavior that we want to test behind some interfaces that would help us mocking the behavior in a unit test. Something like this.
```csharp
public interface IPriceHelper
{
    decimal CalculatePriceTax(decimal originalPrice, decimal stateTax);
    decimal CalalculateFinalPrice(decimal originalPrice, decimal priceTax);
    decimal RoundToTwoDecimals(decimal price);
}
```
And the production code would look like this:
```csharp
public decimal GetFinalPrice(decimal originalPrice, decimal stateTax)
{
    var priceTax = _priceHelper.CalculatePriceTax(originalPrice, stateTax);
    var finalPrice = _priceHelper.CalculateFinalPrice(originalPrice, priceTax);
    return _priceHelper.RoundToTwoDecimals(finalPrice);
}
```
So, we need to test that these operations will be executed:
1. product's original price and state tax are passed to _priceHelper.CalculatePriceTax method:
1. product's original price and result of previous step are passed to _priceHelper.CalculateFinalPrice method:
1. round the result from previous step to two decimals
1. return the result of rounding as output of the GetFinalPrice method

## Testing the behavior with mocking frameworks ##
In the following example, I'm using Moq.
```csharp
private Mock<IPriceHelper> _priceHelper;
private PriceCalculator.LondonStyle.PriceCalculator _sut;

[SetUp]
public void Setup()
{
    _priceHelper = new Mock<IPriceHelper>(MockBehavior.Strict);
    _sut = new PriceCalculator.LondonStyle.PriceCalculator(_priceHelper.Object);
}

[Test]
public void ShouldCalculateFinalPrice()
{
    const decimal originalPrice = 99;
    const decimal stateTax = 7.25m;
    const decimal priceTax = 7.1775m;
    const decimal finalNonRoundedPrice = 106.1775m;
    const decimal finalPrice = 106.18m;

    _priceHelper
        .Setup(x => x.CalculatePriceTax(originalPrice, stateTax))
        .Returns(priceTax);
    _priceHelper
        .Setup(x => x.CalculateFinalPrice(originalPrice, priceTax))
        .Returns(finalNonRoundedPrice);
    _priceHelper
        .Setup(x => x.RoundToTwoDecimals(finalNonRoundedPrice))
        .Returns(finalPrice);

    var price = _sut.GetFinalPrice(originalPrice, stateTax);

    Assert.That(price, Is.EqualTo(finalPrice));

    _priceHelper.VerifyAll();
}
```

Examples from this post can be found on GitHub [here]https://github.com/adimoraret/LondonVsChicago)