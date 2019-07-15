---
layout: post
title:  "Future Orders on Recharge's Theme Engine (TE)"
date:   2019-07-04 10:18:25 -0500
categories: [Recharge, Shopify]
tags: [Recharge, Shopify]
excerpt: "This article compiles the lessons learnt while building a customer portal interface for Vegetable and Butcher using Recharge’s TE, specifically an alternative interface to manage future orders."
---

Recharge recently launched the [Theme Engine](https://rechargepayments.com/features-theme-engine) (in BETA) which allows merchants to customize their customer portal’s UI to their specific needs via a theme editor that resembles Shopify’s own templating engine.

The TE is *“composed of a set of endpoints that provide specific functionality. Each endpoint is associated with a required asset that you can customize to your liking. The endpoint also has a set of objects that can be used to customize the user experience.”*

This article compiles the lessons learnt while building a customer portal interface for [Vegetable and Butcher](vegetableandbutcher.com) using Recharge’s TE, specifically an alternative interface to manage future orders.

## The compromise
While the overall balance of using the TE is really positive with an easy to use web editor and great flexibility to customize most aspects of the customer portal, there’s a considerable compromise we had to make - **customers cannot skip future orders** (other than their next immediate order).

As stated by one of Recharge’s team members via a slack message:

> “On the delivery schedule of the regular customer portal we allow skipping the charges that are in the future, that doesn't even exist in the backend. What we do is, we queue customer action for skip, and our system waits for the time this specific charge is created in the future, and once it does, we skip it.
>
> Such feature doesn't exists on the API or theme editor since you need specific object to exist in order to manipulate with it.”

In addition to this, at the moment of writing this article (June, 2019), the delivery schedule route has very slow response and load times.

With these 2 big restrictions in mind we’re off to find a solution that will let us launch the new UI soon.

## Modifying a customer’s future orders.

There are 2 different elements that hold information about a customer’s future orders, and 2 different endpoints that have the power to modify them.

1. ### Subscriptions / Subscription set next charge date endpoint
   The Subscriptions array consists of `subscription` objects, “individual items customers are receiving on a recurring basis”.

   A `subscription` object has a `next_charge_scheduled_at` property, a string representing the date of the item’s next scheduled order.

   The `subscription.next_charge_scheduled_at` property can be modified via the [subscription set next charge date](https://theme.rechargepayments.com/#subscription-set-next-charge-date) endpoint.


2. ### Delivery Schedule / Skip subscription endpoint
   The Delivery Schedule consists of an array of `shipment` objects, future orders the customer is scheduled to receive.

   A `shipment` object has a `delivery` property, consisting of an array of items to be delivered on the `shipment.date`.

   Each delivery item (let’s name it `deliveryItem`) has a `subscription` property which holds all the info about the specific subscription product, it also has an `is_skipped` property which indicates if the subscription has been skipped for that shipment.

   The `deliveryItem.is_skipped` property is not to be confused with the `deliveryItem.subscription.is_skipped` property, the later refers to the first available shipment of the delivery schedule rather than to it's parent shipment.

   <img src="https://media.giphy.com/media/l378hQDm0Fd41gb4c/source.gif" width="50%">

   Let's visualize it

      ```javascript
      //Delivery Schedule
      [
        {
          "date":"2019-07-06T00:00:00",
          "delivery":[…]
        },
        {
          "date":"2019-07-13T00:00:00",
          "delivery":[
            {
              "is_skipped": true, //2019-07-13
              "subscription":{
                "product_title": "lunch",
                "is_skipped": false, //2019-07-06
                …
              },
              …
            },
            …
          ]
        },
        {
          "date":"2019-07-06T00:00:00",
          "delivery":[…]
        },
        …
      ]
      ```
   This customer has skipped "lunch" for July 13 shipment but is scheduled (not skipped) to get "lunch" on July 6 (first available shipment).

   So breaking down [the compromise](#the-compromise) I mentioned at the beginning of the article:

   - On the **Regular Customer Portal**, customers have the ability to modify the `is_skipped` property for delivery items that belong to **ANY future shipment**.

   - On the **Theme Engine**, customers can only modify the `is_skipped` property for delivery items that belong to the **NEXT available shipment** (`deliverySchedule[0]`).

   The `deliveryItem.is_skipped` property can be modified via the [skip subscription](https://theme.rechargepayments.com/#skip-subscription) endpoint for each of the delivery items on the next shipment.

   One important thing to note is that, although they cannot be directly edited, future skips made prior to moving to the TE are kept and can be displayed on the delivery schedule view.

## It's Complicated...
So we have 2 different endpoints, affecting 2 different properties of 2 different objects. Let’s look into how they relate with each other

- Modifying the `subscription.next_charge_scheduled_at` property (via the subscription set next charge date endpoint) will cause all future skips for that specific subscription product to be reset (`deliveryItem.is_skipped = false`). And I do mean **ALL** of them!

- Skipping a delivery item on the next shipment `deliveryItem.is_skipped` (via the skip subscription endpoint) will cause the `subscription.next_charge_scheduled_at` property to be updated to the next available `shipment.date`

- Skipping/unskipping a delivery item on the next shipment (`deliveryItem.is_skipped`) won't affect subsequent shipments skips.

The 2 endpoints in mention can only be reached from certain pages. Here's an overview of the pros and cons of each.

Page | /subscriptions | /delivery_schedule
--- | --- | ---
Endpoint | subscription set next charge date | skip subscription
Page load time | 3.13 s | 7.92 s
Endpoint response time | 3.77 s | 5.86 s
Can modify next order date | Yes (within the next 25 order cycles) | Yes (within the next 2 order cycles)
Can delay next order by multiple order cycles | Yes | No
Can see future orders skips | No | Yes
Future skips are safe | No | Yes

## Our Solution
So our goal was to provide the customer with a **simple UI** to modify their next order.

Keeping the edit ability on 2 (or more) different pages was against the *simple* part of our goal.

The subscriptions view was a clear choice performance wise and it provided the customer with more flexibility to modify their future orders.

The biggest downside was the inability to display any future skips the customer made before using the TE and the risk of overwritting them. This was the least of all evils for us, and we mitigated by being straighforward about these limitations with the customers -we kept the delivery_schedule as a read-only page where customer could check their future skips. We added a next order editor to the subscriptions page where we always confirm any change by warning the customer about the consequences of it.
