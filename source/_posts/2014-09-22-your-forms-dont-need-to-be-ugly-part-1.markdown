---
layout: post
title: "Your forms don't need to be ugly Part 1"
date: 2014-06-09 21:39:09 +0200
comments: true
categories: android forms styling
---

We all have crated login or signup forms, and quite frankly, most of them are boring, ugly and non responsive to orientation changes.

<!-- more -->

## Why?

Forms are one of those screens in your application where a lot of business rules need to be accomplished. Those rules force us to have
mandatory fields that are not the most appealing ones from a UI point of view. Maybe the country of origin is mandatory field, the group
of age or even the gender.

Without taking into consideration the fact that many apps nowadays force uses to login before actually using it, it may well have a reason for it, those
forms should not frustrate the user nor lead them to exit the app... it should engage them!

For that reason, the login screen should be polished.

Links to

## How to improve them

This is the start of a series of posts on how to improve the look and feel of your forms, suggestions or ideas that could make an impact
in your application and the way how users feel when they see them. There are several mistakes that can be easily fixed and that require just
a bit of polishing taste.

All buttons must have [state list](http://developer.android.com/guide/topics/resources/drawable-resource.html#StateList) and should respect the same padding.

{% img https://lh4.googleusercontent.com/-65P7JghkHg0/U5X0kJ1SzEI/AAAAAAAAEvI/UijRmzcr1r4/w1118-h570-no/device-2014-06-05-155250.jpg %}

Padding between elements is very important, the example above shows a Sign In with Gmail button with wrong padding from [HotelTonight](http://www.hoteltonight.com).

Actions should never take the user to another context. If there is a registration link leading to an external browser you should first think why does that
need to be leading to an external browser in the first place. In case the transition is necessary, there are [several](http://cyrilmottier.com/2014/05/20/custom-animations-with-fragments/)
[examples](http://developer.android.com/training/animation/cardflip.html) on how to create transitions between elements that are appealing to the eye.

In the next part of this series we'll start adding some love to a boring login screen.
