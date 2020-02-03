--- 
layout:             post 
title:              "Testing algorithms or formulas" 
menutitle:          "Testing algorithms or formulas" 
date:               2020-02-02 07:21:00 
category:           CSharp
author:             adimoraret 
language:           EN 
comments:           false 
published:          false
--- 

## London vs Chicago ## 
There are two main schools of thoughts regarding unit testing:
- Chicago style which focuses on results
- London style which focuses on behavior
Let's say we want to calculate final price of a product for a given state.

## Chicago style ##
Most commonly, since according to this style we only care about the result, then we would implement it like this (for simplicity, I'm excluding here typical validation code for given arguments - you can find the full implementation on Github):
```csharp
public decimal GetFinalPrice(decimal price, decimal stateTax)
{
    return price + price * stateTax;
}
```
Here is how a Chicago style unit test would look like:
```csharp
[Test]
public void ShouldCalculateTotalPriceBasedOnValidInputs()
{
    var finalPrice = GetFinalPrice(10, 0.05);
    Assert.That(finalPrice, Is.EqualTo(10.5));
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
Of course we can write then more tests, different inputs, and then we'll address most of the issues. But, it won't be enough. How can we be sure that for evey possible combination, our formula would work?

## London style ##
Since we care about behavior, then we should expect these behaviors:
 - a state tax is applied to the product's original price
 - result is determined by sum of the original price and the result of the above opration
So, we put the behavior that we want to test behind some interfaces that would help us mocking the behavior in an unit test like this.
```csharp
    public interface IPriceHelper
    {
        decimal CalculateProductTax(productPrice, stateTax);
        decimal CalculateFinalPrice(productPrice, productTax);
    }
```
And production code would look like this:
```csharp
public decimal GetFinalPrice(decimal productPrice, decimal stateTax)
{
    var productTax = _priceHelper.CalculateProductTax(productPrice, stateTax);
    var finalPrice = _priceHelper.CalculateFinalPrice(productPrice, productTax);
    return finalPrice;
}
```
So we need to test these:
1. product's price and state tax are passed to _priceHelper.CalculateProductTax method:
1. proudct's price and result of previous step are passed to _priceHelper.CalculateFinalPrice method:
1. output of _priceHelper.CalculateFinalPrice is returned as result of the GetFinalPrice