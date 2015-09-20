---
layout: post
title: "Yahnac meets the Design Support Library"
date: 2015-09-20 09:37:48 +0100
comments: true
categories: material-design, coordinator-layout
---

It's been a while since Yahnac [has received some love](http://www.malmstein.com/blog/2015/04/05/yahnacs-rx-pipeline/). There were several
features I wanted to add and also [some](https://github.com/malmstein/yahnac/issues/25) very
[good](https://github.com/malmstein/yahnac/issues/27) ideas worth exploring.

A fully featured Hacker News client needs to allow the user to **sign in**, **vote** stories and **reply** to comments. Finally, all those features are now available in [Yahnac](https://play.google.com/store/apps/details?id=com.malmstein.yahnac). If you are curious to read about the journey to complete those features and the use of the [Design Support Library](http://android-developers.blogspot.co.uk/2015/05/android-design-support-library.html) to do so please read on.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/top_stories.png %}

<!-- more -->

There are, literally, hundreds of posts about how to use the **Design Support Library**. This is not going to be one of those, my goal here is to explain what I've used and why. As you know, there are a huge number of [Widgets](http://developer.android.com/reference/android/support/design/widget/package-summary.html), [Behaviours](http://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.Behavior.html) and other goodies available in this library. It also means that there are a lot of decisions that need to be made before starting implementing them.

# The glue

It's very easy to get lost in the vast ocean of possibilities, and to me that's the most difficult step to integrate with the Design Support Library. So many options, that is very easy to start adding animations, transitions and effects that are just pretty but are just there for a **mere aesthetic reason and not to bring value to the user**.

When deciding which was going to be the first component I wanted to update it was very simple, the old `NavigationDrawer` was a clear winner.

## NavigationView

There was using was a lot of boilerplate used and several classes in order to display a simple `Drawer`, thanks to the new [NavigationView](https://developer.android.com/reference/android/support/design/widget/NavigationView.html) it's all much easier and cleaner

{% codeblock lang:xml %}
<android.support.design.widget.NavigationView
  android:id="@+id/nav_view"
  android:layout_height="match_parent"
  android:layout_width="wrap_content"
  android:layout_gravity="start"
  android:fitsSystemWindows="true"
  app:headerLayout="@layout/view_drawer_header"
  app:menu="@menu/drawer_view" />
{% endcodeblock %}

Thanks to the `headerLayout` attribute one can define a specific view to be inflated as a header.
Since we are going to show a different header if the user is signed in or not, we can make use of the custom view to decide which layout to inflate.

{% codeblock lang:java %}
@Override
protected void onFinishInflate() {
    super.onFinishInflate();
    ...
    if (loginSharedPreferences.isLoggedIn()) {
      LayoutInflater.from(getContext()).inflate(
        R.layout.view_drawer_header_logged_in, this, true);
    } else {
      LayoutInflater.from(getContext()).inflate(
        R.layout.view_drawer_header_logged_out, this, true);
    }
}
{% endcodeblock %}

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/drawer_signed_in.png %}

## TextInputLayout

In order to reply to comments and vote stories, one need to be signed into [Hacker News](https://news.ycombinator.com/). Even though there is an API, it's not possible to authenticate against HN. Therefore, the only way to do that is to go through the actual website.

**Yahnac does not store any private or sensible information from the user**, the only data stored is the cookie that allows to authenticate the requests. That being said, there is a known issue with this workaround. Every time the cookie is expired (if you sign out from the HN website) you'll need to sign in again. Don't worry, you'll be notified if that happens and you'll be able to sign in again.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/login_screen.png %}

## Toolbar and TabLayout

It was pretty simple to migrate from the old [ActionBar](https://developer.android.com/reference/android/app/ActionBar.html) to the new [Toolbar](https://developer.android.com/reference/android/widget/Toolbar.html). One of those things that make us developers happy. In terms of the new features available, following the trend started a while back by several applications we'll be hiding the toolbar when scrolling the list.

{% codeblock lang:xml %}
<android.support.design.widget.CoordinatorLayout
  android:id="@+id/main_content"
  android:layout_width="match_parent"
  android:layout_height="match_parent">

  <android.support.design.widget.AppBarLayout
    android:id="@+id/appbar"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

    <android.support.v7.widget.Toolbar
      android:id="@+id/toolbar"
      android:layout_width="match_parent"
      android:layout_height="?attr/actionBarSize"
      android:background="?attr/colorPrimary"
      app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
      app:layout_scrollFlags="scroll|enterAlways" />

    <android.support.design.widget.TabLayout
      android:id="@+id/tabs"
      android:paddingLeft="@dimen/appbar_swipe_tabs_left_padding"
      app:tabMode="scrollable"
      android:layout_width="match_parent"
      android:layout_height="wrap_content" />

  </android.support.design.widget.AppBarLayout>

  <android.support.v4.view.ViewPager
    android:id="@+id/viewpager"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layout_behavior="@string/appbar_scrolling_view_behavior" />

</android.support.design.widget.CoordinatorLayout>
{% endcodeblock %}

One can see the reference to [Scroll View Behaviour](https://developer.android.com/reference/android/support/design/widget/AppBarLayout.ScrollingViewBehavior.html) which is by the `ViewPager` which can scroll vertically and support nested scrolling to automatically scroll any `AppBarLayout` siblings.

The new attribute `scroll:flags` is used to determine how the `Toolbar` will behave. In our case, we want the `Toolbar` to hide when scrolling up but appear again as soon as we scroll down.

# Coordinator Layout

A lot has been written about the [CoordinatorLayout](https://developer.android.com/reference/android/support/design/widget/CoordinatorLayout.html), its possibilities and ways of achieving them are out of the scope of this post. However, there is a screen in Yahnac where it made sense to make use of that so called super-powered `FrameLayout`.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/coordinator_comments.gif %}

The Comments screen deserved some love, I wasn't very happy with the actual layour of the comments and it surely could be improved. Also, **adding the vote and reply action to each one of the comments needed a bit more thinking**. In the end, the best solution I came up with was to add action icons to the old comment list items and change the background to improve the reading experience.

After these changes, it's much easier to follow the conversation and much easier to read through a long list of comments.

# Transitions and Animations

As said earlier, it's very easy to fall into the trap of adding too much animations and transitions. It's been hard to restrain myself but I think the outcome is what I wanted to achieve.

In an application like this with no images, there not much space for transitions between master and detail screens. However, one can also **use color as a foundation for these transitions**. That being said, there are only two places where it makes sense to animate the transition between screens. One of them being the transition to the login screen.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/transition_login.gif %}

Both are pretty straight forward Activity transitions, where the content from one screen is animated to the next screen by changing its bounds.

{% codeblock lang:xml %}
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
  <changeBounds />
  <changeTransform />
  <changeClipBounds />
  <changeImageTransform />
  <arcMotion android:minimumHorizontalAngle="45" />
</transitionSet>
{% endcodeblock %}

The animation itself is set in the `Activity` theme.

{% codeblock lang:xml %}
<style name="Theme.HNews.News">
  <item name="android:windowSharedElementEnterTransition">@transition/login_enter</item>
</style>
{% endcodeblock %}

Last but not least, we need to set the `Views` that will form the transition.

{% codeblock lang:java %}
@Override
public void onLoginClicked() {  

final android.util.Pair[] pairs =
  TransitionHelper.createSafeTransitionParticipants(
    activity,
    false,
    new android.util.Pair<>(drawer, VIEW_TOOLBAR_TITLE));

ActivityOptions sceneTransitionAnimation =
ActivityOptions.makeSceneTransitionAnimation(activity, pairs);

final Bundle transitionBundle = sceneTransitionAnimation.toBundle();
Intent commentIntent = new Intent(activity, LoginActivity.class);

ActivityCompat.startActivity(
  activity,
  commentIntent,
  transitionBundle);
}
{% endcodeblock %}

The majority of logic is set in the origin `Activity`, now the only missing piece is to set the `TransitionName` in the target `LoginActivity`

{% codeblock lang:java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    ...
    ViewCompat.setTransitionName(appBar, VIEW_TOOLBAR_TITLE);
    ...
}
{% endcodeblock %}

In terms of animations, I've been particularly interested in the [circular reveal](https://developer.android.com/reference/android/view/ViewAnimationUtils.html#createCircularReveal(android.view.View, int, int, float, float)). When prompting the user for some input it's normal to use `Dialogs` but they are always limiting the amount of space available. In [Hacker News](https://news.ycombinator.com/) is usual to find comments that take more than just 4 lines, if only there was a way of replying to a comment AND have enough space to review it...

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/reveal_reply.gif %}

As you also know, this animation is only available for Lollipop and above. Even though there are several libraries that create this animation before Lollipop, I'm of the opinion that the OS requirements are there for a reason. **Don't back-port them to previous versions since you'll be modifying the default platform behaviour**. Same goes for `Ripples`...

{% codeblock lang:java %}
private void showReplyView() {
    int centerX = (replyFab.getLeft() + replyFab.getRight()) / 2;
    int centerY = (replyFab.getTop() + replyFab.getBottom()) / 2;
    int finalRadius = Math.max(replyView.getWidth(), replyView.getHeight());
    mCircularReveal = ViewAnimationUtils.createCircularReveal(
            replyView, centerX, centerY, 0, finalRadius);

    mCircularReveal.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            super.onAnimationEnd(animation);
            mCircularReveal.removeListener(this);
        }
    });

    mCircularReveal.start();
    replyFab.hide();

    replyView.setVisibility(View.VISIBLE);
    commentsView.setVisibility(View.GONE);
}
{% endcodeblock %}

# Behaviours

One of the most interesting new feature that `CoordinatorLayout` brings is the use of `Behaviours`. One can set dependencies between `Views` and react to the changes in the dependent view.

The Design Library already has several examples of `Behaviours` and those are a very good starting point for us. In this case, the idea was to hide the `FloatingActionBar` when scrolling up the comments list so the reading experience wasn't compromised.

In order to do so, we want the `FAB` to scroll anchored to the `AppBarLayout` and disappear once the its dependent view is no longer visible.

{% img https://raw.githubusercontent.com/malmstein/yahnac/master/art/fab_behaviour.gif %}

For this purpose we are going to extend the [FloatingActionButton.Behavior](https://developer.android.com/reference/android/support/design/widget/FloatingActionButton.Behavior.html). Basically, we only react to vertical scroll and default to the `CoordinatorLayout` scroll. Called when a descendant of the `CoordinatorLayout` attempts to initiate a nested scroll.

Once the `AppBarLayout` is hidden then the `FloatingActionBar` needs to be hidden as well. In order to do so we just have to check the vertical offset and start the out animation. To make things a bit more interesting we'll use a scaling animation to make the button disappear as opposed to simply hiding it.

{% codeblock lang:java %}
@Override
public boolean onStartNestedScroll(...) {

    return nestedScrollAxes == ViewCompat.SCROLL_AXIS_VERTICAL
            || super.onStartNestedScroll(coordinatorLayout,
            child, directTargetChild, target, nestedScrollAxes);
}

@Override
public void onNestedScroll(...) {
  super.onNestedScroll(...);

  if (dyConsumed > 0 && !this.isAnimatingOut
    && child.getVisibility() == View.VISIBLE) {

      animateOut(child);

  } else if (dyConsumed < 0 &&
    child.getVisibility() != View.VISIBLE) {

      animateIn(child);

  }
}

private void animateOut(final FloatingActionButton button) {
    button.setScaleX(1);
    button.setScaleY(1);
    button.animate()
            .scaleX(0)
            .scaleY(0)
            .setInterpolator(animationInterpolator)
            .start();
}

{% endcodeblock %}

# Conclusion

The `Design Support Library` is a huge improvement in the Android development world. It's never been so easy to follow the design patterns that Google puts in place, there are no more excuses now. We have all the tools needed in order to build proper Material Design applications on Android.

In terms of [Yahnac](https://play.google.com/store/apps/details?id=com.malmstein.yahnac). The application is now updated with what you've seen here and it's now a much better Hacker News client.

In case you are interested, I've created a [Google+ Community](https://plus.google.com/communities/108233780766400792163) where you are more than welcome to provide feedback, bug reports or any other related question. Looking forward to it!
