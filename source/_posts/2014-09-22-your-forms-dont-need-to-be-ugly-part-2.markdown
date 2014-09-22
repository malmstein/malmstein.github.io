---
layout: post
title: "Your forms don't need to be ugly Part 2"
date: 2014-06-15 14:26:51 +0200
comments: true
categories: android forms floatinghints styling
---

Last week we [started to talk](http://www.malmstein.com/blog/2014/06/09/your-forms-dont-need-to-be-ugly-part-1/) about how ugly forms can be displayed and raised some issues about it. Today is time to stop ranting and
start contributing.

## Introducing FloatingHints

A while ago thanks to [Chris Banes](https://plus.google.com/+ChrisBanes/posts/5Ejaq51UWGo) and [Cyril Mottier](https://plus.google.com/118417777153109946393/posts/ewdTd7bNw29) there was a buzz around a pattern called Floating Label Layout. I used this approach in
the last project I've been working on and the feedback was very good. Therefore, I'm releasing this small library called [FloatingHints](https://github.com/malmstein/FloatingHints) which
you could use if you want to add a bit of styling to those ugly forms.

The snapshot version is **already available** at Sonatype while the release version will be soon released at Maven Central.

<!-- more -->

## If you wouldn't use that form, why would the user?

In the previous post from the series we saw one of the most common form structures. A couple of EditText where the user will write the
user and password and a button used to start the Login action.

With the addition of the Floating Label pattern one can have a much nicer and clear screen. No one likes to fill these forms, not even the users
excited about using your service... so why don't we encourage them to do so?

{% img https://lh4.googleusercontent.com/-65P7JghkHg0/U5X0kJ1SzEI/AAAAAAAAEvI/UijRmzcr1r4/w1118-h570-no/device-2014-06-05-155250.jpg %}

## Custom Animations

One of the key features of FloatingHints is that the animations which are in charge of showing or hiding the floating hint are fully customisable.
By default, the hint will be shown using a **Slide in to Top** and will be hidden using a **Slide out to Bottom**

{% img center https://raw.githubusercontent.com/malmstein/FloatingHints/master/art/floating.gif %}

This can be easily customised via XML attributes.

{% codeblock lang:xml %}
  <com.malmstein.floatinghints.FloatingHintEditTextLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:floatLayoutInAnimation="@anim/abc_fade_in"
    app:floatLayoutOutAnimation="@anim/abc_fade_out">

    <EditText
      style="@style/FloatLabel.Form.Text"
      android:id="@+id/float_login_email"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:hint="@string/hint_email"
      android:inputType="textEmailAddress" />

  </com.malmstein.floatinghints.FloatingHintEditTextLayout>
{% endcodeblock %}

The result of using **Fade In** and **Fade Out** is pretty good too!

{% img center https://raw.githubusercontent.com/malmstein/FloatingHints/master/art/floating_fade.gif %}

## Customise the color of the hint

The `FloatingHintEditTextLayout` uses the attribute `floatLayoutTextAppearance` to change the [TextAppearance](http://developer.android.com/reference/android/widget/TextView.html#attr_android:textAppearance)
of the Label.

**activity_demo.xml**
{% codeblock lang:xml %}
  <com.malmstein.floatinghints.FloatingHintEditTextLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:floatLayoutInAnimation="@anim/abc_fade_in"
    app:floatLayoutOutAnimation="@anim/abc_fade_out"
    app:floatLayoutTextAppearance="@style/FloatLabel.Label">

    <EditText
      style="@style/FloatLabel.Form.Text"
      android:id="@+id/float_login_email"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:hint="@string/hint_email"
      android:inputType="textEmailAddress" />

  </com.malmstein.floatinghints.FloatingHintEditTextLayout>

{% endcodeblock %}

**styles.xml**
{% codeblock lang:xml %}
  <style name="FloatLabel.Label" parent="android:TextAppearance.Small">
    <item name="android:textColor">@color/float_edit_text_layout_color</item>
    <item name="android:textSize">@dimen/text_xs</item>
    <item name="android:textStyle">bold</item>
  </style>
{% endcodeblock %}

**float_edit_text_layout_color.xml**
{% codeblock lang:xml %}
<selector xmlns:android="http://schemas.android.com/apk/res/android">
  <item android:color="@color/font_yellow" android:state_activated="true" />
  <item android:color="@color/font_grey" />
</selector>
{% endcodeblock %}

This way we've just changed the text appearance of the label to yellow.

{% img center https://raw.githubusercontent.com/malmstein/FloatingHints/master/art/floating_fade_yellow.gif %}

## Wrapping up

The [demo](https://github.com/malmstein/FloatingHints/tree/master/app/src/main) project from [FloatingHints](https://github.com/malmstein/FloatingHints) contains a wrap up of all the ideas in this post.
Check it out, fork it and don't hesitate in giving some feedback. It will be very much appreciated.

Stay tuned for the next post of the series!
