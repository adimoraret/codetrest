---
layout:             post
title:              "How to avoid if/else branches"
menutitle:          "How to avoid if/else branches"
date:               2019-10-07 07:21:00
category:           CSharp
author:             adimoraret
language:           EN
comments:           false
published:          false
---
## What is the problem with if/else ##
Adding if/else branches increases the cyclomatic complexity of the code. A high cyclomatic complexity is hard to read no matter how well the code is written. Consider the following example.
```csharp
    public decimal RoundAssetValue(Coin coin, decimal value){
        if (coin == Coin.FIAT) 
        {
            return Math.Round(value, 2, MidpointRounding.AwayFromZero)
        }
        else if (coin == Coin.BTC)
        {
            return Math.Round(value, 8, MidpointRounding.AwayFromZero) 
        }
        else if (coin == Coin.XRP)
        {
            return Math.Round(value, 6, MidpointRounding.AwayFromZero) 
        }
    }
```
The above code has a cyclomatic complexity of 2. Every time an if/else is added, cyclomatic complexity is increased with one. When there are no ifs, or looping instructions, this complexity is equal to 1.
Of course there are some other problems with this: it violates Open/Closed principle because every time we need to introduce a coin we'll need to change this function. So this function will not be closed to modifications.

Ways to avoid this:
### Using a dictionary ###
There are some aproaches to use dictionary. Personally, I'm not a fan of this.
```csharp
   //Define our dictionary:
    var coinRoundings = new Dictionary<Coin, decimal>
    {
       {Coin.FIAT, (x) => Math.Round(x, 2, MidpointRounding.AwayFromZero) },
       {Coin.BTC, (x) => Math.Round(x, 8, MidpointRounding.AwayFromZero) },
       {Coin.XRP, (x) => Math.Round(x, 6, MidpointRounding.AwayFromZero) },
    } 
    //Result
    const myTestCoin = Coin.BTC;
    var result = coinRoundings[myTestCoin].call(9.304);
```
There are a lot of disadvantages with this approach:
1. It is hard to read
1. It introduces reflection code
1. It is not scalable.
1. It violates Open/Closed principle

### Using inheritance ###
Using template design pattern we could create something like this.
```csharp
    public abstract class CoinRounding
    {
        public abstract int NumberOfDecimals { get; }
        public decimal Round(decimal value) => Math.Round(value, NumberOfDecimals, MidpointRounding.AwayFromZero);
    }
    public abstract class FiatRounding
    {
        public override int NumberOfDecimals { get; } = 2;
    }
    public abstract class BtcRounding
    {
        public override int NumberOfDecimals { get; } = 8;
    }
    public abstract class XrpRounding
    {
        public override int NumberOfDecimals { get; } = 6;
    }
    //usage
    const myTestCoin = Coin.BTC;
    CoinRounding coinRounding = coinRoundingFactory.create(myTestCoin);
    const result = coinRounding.Round(123.8405890348509);
```
