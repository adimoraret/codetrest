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
    var promotion = _promotionRepository.GetPromotion(discountCode);
    if (promotion.IsValid())
    {
        _checkoutCart.ApplyPromotion(promotion);
    }
    else
    {
        _notificationService.SendWarning(`Invalid discount code: ${discountCode}`);
    }
```
The above code has a cyclomatic complexity of 2. It is not too high, but it's easy to imagine few extra verifications which will increase it. How can we re-write this so we'll get to a cyclomatic complexity of 1?

### Using a dictionary ###
We could use a dictionary
```csharp
    var promotion = _promotionRepository.GetPromotion(discountCode);
    var promotionActions = new Dictionary<bool, Action>
    {
        {true, () => _checkoutCart.ApplyPromotion(promotion)},
        {false, () =>  _notificationService.SendWarning(`Invalid discount code: ${discountCode}`);}
    };
    promotionActions[promotion.IsValid()].Call();    
```
But, what if there are 3,4,5 or more conditions? The dictionary from above would not help us in the following example, well at least not in that form. 
```csharp
    var promotion = _promotionRepository.GetPromotion(discountCode);
    if (promotion.IsValid())
    {
        _checkoutCart.ApplyPromotion(promotion);
    }
    else if (promotion.IsExpired())
    {
        _notificationService.SendWarning(`Discount code: ${discountCode} has expired`);
    }
    else 
    {
        _notificationService.SendWarning(`Invalid discount code: ${discountCode}`);
    }
```

### Using the visitor pattern ###
