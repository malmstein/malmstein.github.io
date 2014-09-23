---
layout: post
title: "Using Gestures to control the Media Player"
date: 2014-09-21 20:24:59 +0200
comments: true
categories: fenster, gestures, mediaplayer
---


In the previous post we explained a way of displaying a [customised Media Player controller](http://www.malmstein.com/blog/2014/08/09/how-to-use-a-textureview-to-display-a-video-with-custom-media-player-controls),
which as we saw, it's very useful if we need customised actions or UI for that controller.

This time we want to go a step forward and introduce a new way of interacting with the Media Player.
The idea is to use gestures in order to lower or raise the volume, seek forward and backwards
and to skip the video for 30 seconds.

I'm a big fan of [MX Player](https://play.google.com/store/apps/details?id=com.mxtech.videoplayer.ad) and
always wanted to implement the gesture detection that they introduced.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/fenster_gesture.gif %}

<!-- more -->

In order to achieve this, the first thing we need to understand is how the gestures
are detected and how we can use them in our benefit.

When one or more fingers are placed on the screen, the callback `onTouchEvent()` is triggered on
the View that received the touch events. What's important here is to understand that a gesture
is a sequence of touch events and each one of them will fire `onTouchEvent()`.

A gesture starts when the user first touches the screen, continues as the system tracks the
position of the user's finger(s), and ends by capturing the final event of the user's fingers
leaving the screen. Throughout this interaction, the [MotionEvent](http://developer.android.com/reference/android/view/MotionEvent.html)
delivered to `onTouchEvent()` provides the details of every interaction.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/basic_multitouch.gif %}

## Understanding MotionEvent

Android uses `MotionEvent` to report movements, providing an action and a set of axis values. **Each action
describes the state change occurred**, such as pointer going down, up or moving. Another very important
aspect of `MotionEvent` is that not only describes one movement but can also **describe multi-touch events**.
This is particularly important if we are trying to identify multi-touch gestures such as
scrolling with two fingers or two finger double tap.

## Gesture Detector

Now that we know how events are described on Android, it's time to understand how gestures are
detected and how to apply it to our purpose. For that matter, Android provides the
[GestureDetector](http://developer.android.com/reference/android/view/GestureDetector.html)

There are two different ways of intercepting gestures on Android. One can use
the [SimpleGestureListener](http://developer.android.com/reference/android/view/GestureDetector.SimpleOnGestureListener.html) class,
which wraps around the interfaces [OnGestureListener](http://developer.android.com/reference/android/view/GestureDetector.OnGestureListener.html)
and [OnDoubleTapListener](http://developer.android.com/reference/android/view/GestureDetector.OnDoubleTapListener.html).
Think about it as a convenience class to extend when you only want to listen for a subset
of all the gestures.

The other approach is to implement the interfaces that are more interesting to you, since
we here to play, that's the one we’ll follow.

As you can see there are several events that we can intercept:

   * `onDown()` : Notified when a tap occurs with the event that triggered it. This will be triggered immediately for every down event. All other events should be preceded by this.
   * `onShowPress()`: The user has performed a down {@link MotionEvent} and not performed a move or up yet.
   * `onSingleTapUp()`: Notified when a tap occurs with the up that triggered it.
   * `onScroll()`: Notified when a scroll occurs with the initial on down and the current move. The distance in x and y is also supplied for convenience.
   * `onLongPress()`: Notified when a long press occurs with the initial on down that trigged it.
   * `onFling()`: Notified of a fling event when it occurs with the initial on down and the matching up. The calculated velocity is supplied along the x and y axis in pixels per second.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/basic_gesture.gif %}

## The Gesture View

Now that we have all the tools and we understand how this whole thing works, let's get our hands
dirty. Our approach will be to add a `View` to our `PlayerController`, this view will handle
all the gestures and delegate the interaction of those to the `PlayerController`.

All we need to do here is to send all the intercepted events coming from the
`FensterGestureControllerView` to the `GestureDetector` via the `onTouchEvent()`.

{% codeblock lang:java %}
@Override
public boolean onTouchEvent(MotionEvent event) {

    gestureDetector.onTouchEvent(event);
    return super.onTouchEvent(event);
}
{% endcodeblock %}

As we said earlier, the `GestureDetector` is the one in charge of detecting all these events, in
combination with the 'GestureListener'

## The Gesture and Events listeners

We are only interested in some of the gestures that can be detected, in our case it's all described
in the following interface

{% codeblock lang:java %}
public interface FensterEventsListener {

    void onTap();

    void onHorizontalScroll(MotionEvent event, float delta);

    void onVerticalScroll(MotionEvent event, float delta);

    void onSwipeRight();

    void onSwipeLeft();

    void onSwipeBottom();

    void onSwipeTop();
}
{% endcodeblock %}

Once we detect one of these events, the `MediaPlayerController` will react by changing the
progress bar, raising the volume or skipping. Pretty straight forward right? Let's actually see
where the magic happens, the gesture detection.

As said earlier, the [OnGestureListener](http://developer.android.com/reference/android/view/GestureDetector.OnGestureListener.html)
provides the interfaces we are interested in, let's start by detecting the **Horizontal** and **Vertical** scrolls.

{% codeblock lang:java %}
    @Override
    public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {

        Log.i(TAG, "Scroll");

        float deltaY = e2.getY() - e1.getY();
        float deltaX = e2.getX() - e1.getX();

        if (Math.abs(deltaX) > Math.abs(deltaY)) {
            if (Math.abs(deltaX) > SWIPE_THRESHOLD) {
                listener.onHorizontalScroll(e2, deltaX);
                if (deltaX > 0) {
                    Log.i(TAG, "Slide right");
                } else {
                    Log.i(TAG, "Slide left");
                }
            }
        } else {
            if (Math.abs(deltaY) > SWIPE_THRESHOLD) {
                listener.onVerticalScroll(e2, deltaY);
                if (deltaY > 0) {
                    Log.i(TAG, "Slide down");
                } else {
                    Log.i(TAG, "Slide up");
                }
            }
        }
        return false;
    }
{% endcodeblock %}

The `onScroll()` interface provides the initial on down `MotionEvent` and the current one, using
the axis values we can detect the direction towards where the scrolling movement is going
and calculate the delta of the movement.

Note the threshold value, that needs to be there in order to filter movements. We don't want to
be confused with small movements or flings, only the gestures which have moved a certain
distance can be considered as scrolls.

It's also important to understand the difference between scrolls and flings since they are also
going to be treated differently. A **scroll is a continuos movement** like the one you'll use to
navigate through a timeline, as opposed to a **fling which is intentionally a short and fast movement**
which one will use to navigate horizontally through a view pager.

{% codeblock lang:java %}
    @Override
    public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
        // Fling event occurred.  Notification of this one happens after an "up" event.
        Log.i(TAG, "Fling");
        boolean result = false;
        try {
            float diffY = e2.getY() - e1.getY();
            float diffX = e2.getX() - e1.getX();
            if (Math.abs(diffX) > Math.abs(diffY)) {
                if (Math.abs(diffX) > SWIPE_THRESHOLD && Math.abs(velocityX) > minFlingVelocity) {
                    if (diffX > 0) {
                        listener.onSwipeRight();
                    } else {
                        listener.onSwipeLeft();
                    }
                }
                result = true;
            } else if (Math.abs(diffY) > SWIPE_THRESHOLD && Math.abs(velocityY) > minFlingVelocity) {
                if (diffY > 0) {
                    listener.onSwipeBottom();
                } else {
                    listener.onSwipeTop();
                }
            }
            result = true;

        } catch (Exception exception) {
            exception.printStackTrace();
        }
        return result;
    }
{% endcodeblock %}

There is a new constant introduced here, the `minFlingVelocity`. Since the fling is considered to
be a fast movement, we need to make sure that the motion event velocity is fast enough.

How do we get these constant values? Android conveniently has a  [ViewConfiguration]("http://developer.android.com/reference/android/view/ViewConfiguration.html")
class with standard constants used in the UI for timeouts, sizes, and distances.

## The Controller View

Ultimately, it will be the player controller the guy who will react to the events.
That's why we delegate all the responsibility to the gesture listener. We attach the listener
from the player controller to the listener like this:

{% codeblock lang:java %}
    public void setFensterEventsListener(FensterEventsListener listener){
        gestureDetector = new GestureDetector(getContext(),
                          new FensterGestureListener(listener,
                              ViewConfiguration.get(getContext())));
    }
{% endcodeblock %}

## Acting on the Media Player

Finally, we have all the pieces together. We are now able to detect when the user is touching
the screen and we can also detect the type of gesture that is happening.

{% codeblock lang:java %}
    @Override
    public void onHorizontalScroll(MotionEvent event, float delta) {
        updateVideoProgressBar(delta);
    }

    private void updateVideoProgressBar(float delta) {
        mSeekListener.onProgressChanged(mProgress,
            extractHorizontalDeltaScale(delta, mProgress), true);
    }
{% endcodeblock %}

For this first attempt we will only act on the video progress bar, which is a pretty straigh
forward thing to do since we already have a `seekListener` which acts on the media player. The
only thing we need to do is scaling the user movement compared to the [SeekBar]("http://developer.android.com/reference/android/widget/SeekBar.html").

The interface is giving us a **delta**, which is the amount of pixels of the gesture. We are
taking as reference the whole `View` width, that way when the user scrolls the whole width of
the screen it will also seek through the whole video in a 1:1 scale. The last thing we
need to take into consideration the **MAX** parameter applied to the `SeekBar`.

{% codeblock lang:java %}
    private int extractDeltaScale(int availableSpace, float deltaX, SeekBar seekbar){

        int x = (int) deltaX;
        float scale;
        float progress = seekbar.getProgress();
        final int max = seekbar.getMax();

        if (x < 0) {
            // scrolling back
            scale = (float) (x) / (float) (max - availableSpace) ;
            progress = progress - (scale * progress);
        } else {
            // scrolling forwards
            scale = (float) (x) / (float) availableSpace;
            progress += scale * max;
        }

        return (int) progress;
    }
{% endcodeblock %}

Once this is all computed we can now scroll back and forth across all the video timeline.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/fenster_gesture.gif %}

## Further reading

If you want to experience further in detecting gestures and multitouch events, don’t
hesitate in importing two projects from the SDK samples: `BasicGestureDetect` and `BasicMultitouch`.
Yeah, they are basic, but it establishes the foundations of what we are going to do here.

There is also a pretty good training document about [Gesture Detection](http://developer.android.com/training/gestures/detector.html) in
the Android Developer website.

## Fenster Library update

In the next post we'll use the same techniques to act on several other aspects of the
`MediaPlayer`. In the meantime, the [Fenster](https://github.com/malmstein/fenster) library has
been updated with all the parts of this post and the demo project also contains an example of
gesture detection against the `MediaPlayer`.

As always, all feedback received is more than welcome!
