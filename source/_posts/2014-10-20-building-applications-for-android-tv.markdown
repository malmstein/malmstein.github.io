---
layout: post
title: "Building applications for Android TV"
date: 2014-10-21 00:19:05 +0100
comments: true
categories: androidtv lollipop
---

By now you all have heard about [the new Android release](http://developer.android.com/about/versions/lollipop.html) and almost all of you have
played with new API and probably experimented with [Material Design](http://www.google.com/design/spec/material-design/introduction.html).

I have to say though, that one of the things that I'm most excited about is [Android TV](http://developer.android.com/design/tv/index.html). I was very much into it [before it was cool](http://www.google.com/tv/) but never got the chance to actually get hands on. Happily this has changed now and I'll be writing a series of posts about developing for Android TV, how to start, how to make your existing app compatible and how to make the most of it.

If you are interested, please read on!

{% img https://raw.githubusercontent.com/malmstein/AndroidTVExplorer/master/art/intro.gif %}

<!-- more -->

Building applications for Android TV is not just developing and designing for an extremely big tablet. Yes, it is
a much bigger display but that's not the point. **The interaction with the device is completely different compared to a phone**, you are not going
to scroll through a 55 inches TV, are you? Same goes for the use cases, so far no one has been spotted reading the news in the underground on a [Sony Bravia](http://www.engadget.com/2014/06/25/android-tv-is-coming-to-sony-sharp-and-philips-tvs-next-year/).

# Support Libraries

To make things a bit easier for us, Google has created a [Leanback Support Library](http://developer.android.com/tools/support-library/features.html#v17-leanback) which contains APIs to support building user interfaces on TV devices. It provides a number of important widgets for TV apps which we'll make use in this project.

{% codeblock lang:groovy %}
dependencies {
  compile 'com.android.support:recyclerview-v7:21.0.+'
  compile 'com.android.support:leanback-v17:21.0.+'
  compile 'com.android.support:appcompat-v7:21.0.+'
}
{% endcodeblock %}

# Demo project: The Android TV Explorer

It's always easier to show and share while building stuff so I'm creating this example project called [Android TV Explorer](https://github.com/malmstein/AndroidTVExplorer) where all the knowledge from this series will be pushed.

By the end of this series the project will be a **great starting point** for anyone interested in developing a media library application for Android TV.

{% img https://raw.githubusercontent.com/malmstein/AndroidTVExplorer/master/art/home.png %}

# Creating a library layout using the Browse Fragment

One of the widgets that the `Leanback Support Library` provides is the [BrowseFragment](http://developer.android.com/reference/android/support/v17/leanback/app/BrowseFragment.html).
A fragment for creating leanback browse screens, it is composed of a `RowsFragment` and a `HeadersFragment`.

As you can see in the library screen above, the header corresponds to the list on the left and the rows to the grid on the right. `BrowseFragment` allows us to customise several actions and some UI elements, like changing the default visibility of the `HeadersFragment` or changing the behaviour of the transition when entering one of the rows.

For the purpose of this example we are not going to change anything today, indeed we are going to
take a closer look at **how the items are displayed**.

{% codeblock lang:java %}
private void setupUIElements() {
  setTitle(getString(R.string.browse_title));
  setHeadersState(HEADERS_ENABLED);
  setHeadersTransitionOnBackEnabled(true);
  setBrandColor(getResources().getColor(R.color.fastlane_background));
  setSearchAffordanceColor(getResources().getColor(R.color.search_opaque));
}
{% endcodeblock %}

There several helper methods that the `BrowseFragment` exposes that make it very simple to customise its look and feel. For example, `setBrandColor()` and `setSearchAffordanceColor()` allow us to change the background of the `HeadersFragment` and the color of the rounded search icon.

# Presenter

`BrowseFragment` renders the elements of its [ArrayObjectAdapter](http://developer.android.com/reference/android/support/v17/leanback/widget/ArrayObjectAdapter.html) as a sequence of rows and a vertical list. This `ObjectAdapter` makes use of a [Presenter](http://developer.android.com/reference/android/support/v17/leanback/widget/Presenter.html), concept which those familiar with the [Model-view-presenter](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter) will quickly understand.

A `Presenter` is used to generate `Views` and bind `Objects` to them on demand. It is closely related to concept of an `RecyclerView.Adapter`, but is non position-based.

{% codeblock lang:java %}  
@Override
public void onLoadFinished(Loader<HashMap<String, List<Movie>>> arg0, HashMap<String, List<Movie>> data) {
  mRowsAdapter = new ArrayObjectAdapter(new ListRowPresenter());
  CardPresenter cardPresenter = new CardPresenter();
  int i = 0;

  for (Map.Entry<String, List<Movie>> entry : data.entrySet()) {
      ArrayObjectAdapter listRowAdapter = new ArrayObjectAdapter(cardPresenter);
      List<Movie> list = entry.getValue();

      for (int j = 0; j < list.size(); j++) {
          listRowAdapter.add(list.get(j));
      }
      HeaderItem header = new HeaderItem(i, entry.getKey(), null);
      i++;
      mRowsAdapter.add(new ListRow(header, listRowAdapter));
  }

  setAdapter(mRowsAdapter);
}
{% endcodeblock %}

We are using a `CardPresenter` which will **display all the movies from each category into cards**. The elements of the `BrowseFragment` adapter must be subclasses of 'Row', that's the reason why the `mRowsAdapter` uses the `ListRowPresenter`. Again, this is just a wrapper around normal fragments and its goal is to make things easy to develop from scratch. Obviously, one can make use of them or not but for the purpose of this project we'll stick to the simplicity... for now!

# Setting up event listeners

There are two exposed `Listeners` in the `BrowseFragment`. It's very interesting to see
how the selection and click are two different events, the reason why is very simple and you'll get
to understand it after reading the code.

{% codeblock lang:java %}  
protected OnItemViewSelectedListener getDefaultItemSelectedListener() {
  return new OnItemViewSelectedListener() {
    @Override
    public void onItemSelected(Presenter.ViewHolder viewHolder, Object item, RowPresenter.ViewHolder                  viewHolder2, Row row) {
      mBackgroundURI = ((Movie) item).getBackgroundImageURI();
      updateBackgroundPicture();
    }
  };
}

protected OnItemViewClickedListener getDefaultItemClickedListener() {
  return new OnItemViewClickedListener() {
    @Override
    public void onItemClicked(Presenter.ViewHolder viewHolder, Object item, RowPresenter.ViewHolder viewHolder2, Row row) {
      Toast.makeText(getActivity(), movie.getTitle(), Toast.LENGTH_LONG);
    }
  };
}
{% endcodeblock %}

When a video is selected we can use the [BackgroundManager](https://developer.android.com/reference/android/support/v17/leanback/app/BackgroundManager.html) to update the background picture.

# Background Manager

This is a complete different approach to what a mobile or tablet application would do but in my honest opinion makes a lot of sense. **Android TV supports background image continuity between multiple `Activities`**.
We are used to have transitions between all the `Activities` but this can be confusing in bigger form factors and when there is not a context switch, making use of the `Background Manager` we can deliver smoother transitions that won't disturb the user when looking at a big screen.

{% codeblock lang:java %}  
protected void updateBackground(Drawable drawable) {
  BackgroundManager.getInstance(getActivity()).setDrawable(drawable);
}
{% endcodeblock %}

## Tip

Setting up the environment might be a bit tricky when talking about Android TV. Some of you may have an
[ADT Developer Kit](http://developer.android.com/tv/adt-1/index.html) but if you don't you'll need to create
an emulator to start with (or wait for the brand new Nexus Player). It's pretty straight forward to [create an emulator](http://developer.android.com/training/tv/start/start.html#run) but it's very helpful to activate
[hardware acceleration](http://developer.android.com/tools/devices/emulator.html#acceleration) in order to
have better performance in the Android TV emulator.

# Show me the code

You'll find the entire project on [Github](https://github.com/malmstein/AndroidTVExplorer) and I'll be adding
more features along with all the blog posts of this series. As always, all ~~trolling~~ feedback is more than welcome!
