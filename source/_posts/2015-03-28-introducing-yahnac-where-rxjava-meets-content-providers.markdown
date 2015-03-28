---
layout: post
title: "Introducing Yahnac: Where RxJava meets Firebase and Content Providers"
date: 2015-03-28 09:45:57 +0000
comments: true
categories: yahnac hackernews contentprovider rxjava firebase
---

What's **Yahnac** you might ask? Yet another [Hacker News](https://news.ycombinator.com/) client, because there are never enough Hacker News clients out there!

For those who don't know, [Hacker News](https://news.ycombinator.com/) is a social news website focusing on computer science and entrepreneurship and it is run by the startup incubator [Y Combinator](https://www.ycombinator.com/). In general, content that can be submitted is defined as "anything that gratifies one's intellectual curiosity".

Not so long ago Y Combinator [announced](http://blog.ycombinator.com/hacker-news-api) a long awaited [API for Hacker News](https://github.com/HackerNews/API). Although it was pretty basic and simple, the most exciting piece of news to me was that the API was built using [Firebase](https://www.firebase.com/).

In terms of features, **Yahnac allows you to read all Hacker News content from Hacker News from Top Stories to Jobs**. On top of that, **you'll be able to add Bookmarks** and keep them as long as you like.

If you are interested in knowing how **Yahnac** has been built, please do Read on!

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/top_stories.png %}

<!-- more -->

# A confession to make

Doctor, I have to confess. I like [Loaders](http://developer.android.com/guide/components/loaders.html), [Content Providers](http://developer.android.com/guide/topics/providers/content-providers.html) and their boilerplate. There is no point arguing if they are the best solution for everything, they are not.

A few months ago [RxJava](https://github.com/ReactiveX/RxJava) started to become popular amongst [Novoda](http://novoda.com/). Applications like [Digital Concert Hall](https://play.google.com/store/apps/details?id=com.novoda.dch) and [The Sun](https://play.google.com/store/apps/details?id=uk.co.thesun.mobile) have been developed with reactive approach and the results are amazing.

Also, I'm lucky enough to have people like [Benjamin Augustin](http://uk.droidcon.com/2014/sessions/rx-fy-all-the-things/), [Volker Leck](https://twitter.com/devisnik) and [Antonio Bertucci](https://twitter.com/mr_archano) around the office, who strongly believe in the benefits of reactive programming and it's applications to Android.

Following these principles I started a journey to wrap all these Framework specific solutions together with `Rx` and `Firebase`, a very different approach to conventional Rest APIs.

# Architecture

The principle is very simple, it doesn't matter how the data is retrieved from the network since all I really care is to store it into the database in a meaningful way and display it as quick as possible. These very distinct responsibilities can be handled by `Rx` on the networking layer, [SQlite](https://www.sqlite.org/) in the database layer and `Loaders` in the UI layer.

Leveraging all the network and data manipulation onto `Rx` together with `Firebase` makes a lot of sense. `Firebase` data is [retrieved by attaching an asynchronous listener to a Firebase reference](https://www.firebase.com/docs/android/guide/retrieving-data.html). The listener will be triggered once for the initial state of the data and again anytime the data changes.

Based on the current implementation of the API, one needs to retrieve to 501 `Firebase` instances in order to display a list of [500 top stories](https://github.com/HackerNews/API#new-and-top-stories). One call will retrieve the id of all the items in the Top Stories page, then one needs to do another call for each one of those id and retrieve the data from the news item.

`Rx` seems like a perfect candidate to solve all this, it's reactive approach allows to deal with all the recursive calls and threading seamlessly. It also allows to manipulate that data and provide with the output that the database layer needs. The output of this will be [ContentValues](http://developer.android.com/reference/android/content/ContentValues.html), which is what a `ContentProvider` needs.

On the UI layer, once we are using `ContentProvider` it's very easy to decide how to display the content. [CursorLoader](http://developer.android.com/reference/android/content/CursorLoader.html) allows you to retrieve a [Cursor](http://developer.android.com/reference/android/database/Cursor.html) with the desired data, it all made sense!

There was only one step more to be take, and that was to solve the incompatibility of [RecyclerView](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html) and `Cursor`, in following post I'll deep dive into the actual implementation details and issues, stay tuned if you are interested.

# Supporting different form factors

I had several doubts regarding the UI and how it would look like in different form factors. At the beginning I thought about using a [Multi Pane](http://developer.android.com/design/patterns/multi-pane-layouts.html) but after using the app for a while something didn't feel right.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/tablet_old_ui.png %}

The navigation wasn't very clear on landscape so I decided to use a `Staggered Grid` like the one [Etsy](https://codeascraft.com/) made so popular back in the days.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/tablet_top_stories.png %}

Same goes for smartphones, the UI will adapt from one column to two.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/smartphone_landscape.png %}

# We live in a Material world

No decent app nowadays should be shipped without [Material Design principles](http://www.google.com/design/spec/material-design/introduction.html) and after [talking](https://speakerdeck.com/malmstein/what-material-design-means-to-android) and [presenting](https://speakerdeck.com/malmstein/material-animations) about it I couldn't be different.

The majority of articles are shown in a WebView, that does not leave a lot of space for transitions and fancy animations. The application has no pictures therefore beautiful [Palette](https://developer.android.com/reference/android/support/v7/graphics/Palette.html) can't be added, not the best case scenario is it?

However, Material is not just animations and colors. Several aspects like **typograhpy**, **spacing** and **ripples** can easily be applied making a big difference.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/comments.png %}

Another of the features that all Material applications show these days is the **quick return pattern**. It's a great idea and allows the user to enjoy more content while scrolling.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/yahnac.gif %}

# Right, how do I get it?

[![Get it on Google Play](https://raw.githubusercontent.com/malmstein/yahnac/master/art/play.png)][1]

# There's more to come

The application is not finished, there are several feature that I'd like to add like being able to publish a news item or replying to comments. Do yo fancy contributing? Please do! The [source code is available in GitHub](https://github.com/malmstein/yahnac) and all PRs are welcome.

[1]: https://play.google.com/store/apps/details?id=com.malmstein.yahnac
