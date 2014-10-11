---
layout: post
title: "All your gestures belong to Fenster"
date: 2014-10-11 14:12:51 +0100
comments: true
categories: fenster, gestures, mediaplayer, seekbar
---

In the [previous post](www.malmstein.com/blog/2014/09/21/control-the-media-player-using-gestures/)
we showed how to implement some basic gestures and control the `MediaPlayer`.

Today we are going a step further by implementing multitouch gestures. So far we can
control the progress of a video but that's not enough. If we want full control over the
`MediaPlayer` we need to be able to change the volume, adjust the brightness and skip the
video forward and backwards.

{% img https://raw.githubusercontent.com/malmstein/fenster/master/art/multitouch.gif %}

<!-- more -->

The `MediaPlayerController` source code is getting bigger and bigger so it's very important to keep
things as reusable and clean as possible. For that reason, tasks like update the volume
or adjusting the brightness can be done from their own components, in these case, [SeekBar](http://developer.android.com/reference/android/widget/SeekBar.html).

With these new `Views`, one will be able to just import and use these small controllers without having
to import the whole `MediaPlayerController`, just trying to make things easy here!

# Volume SeekBar

Taking advantage of the gestures, we are also going to update the volume... how do we do that?
There is now a new component in [Fenster](https://github.com/malmstein/fenster) called [VolumeSeekBar](https://github.com/malmstein/fenster),
a `SeekBar` which updates the volume of the [AudioManager](http://developer.android.com/reference/android/media/AudioManager.html).

{% codeblock lang:java %}
public void initialise(final Listener volumeListener) {
  this.audioManager = (AudioManager) getContext().getSystemService(Context.AUDIO_SERVICE);
  this.volumeListener = volumeListener;

  this.setMax(audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC));
  this.setProgress(audioManager.getStreamVolume(AudioManager.STREAM_MUSIC));
  this.setOnSeekBarChangeListener(volumeSeekListener);
}
{% endcodeblock %}

Here you can see the public method to initialise the `View`, the only thing we need to pass is
the `Listener` which will notify when the view is being dragged.

The `AudioManager` is the object which is in charge of the volume, `setStreamVolume()` is the
method which is exposed for that matter. Since we want to update them based on the changes
on the `SeekBar`, the `SeekBarChangeListener` is the way we want to go.

{% codeblock lang:java %}
private final OnSeekBarChangeListener volumeSeekListener = new OnSeekBarChangeListener() {
  @Override
  public void onProgressChanged(SeekBar seekBar, int vol, boolean fromUser) {
    audioManager.setStreamVolume(AudioManager.STREAM_MUSIC, vol, 0);
  }

  @Override
  public void onStartTrackingTouch(SeekBar seekBar) {
    volumeListener.onVolumeStartedDragging();
  }

  @Override
  public void onStopTrackingTouch(SeekBar seekBar) {
    volumeListener.onVolumeFinishedDragging();
  }
};
{% endcodeblock %}

As you can see, we are also notifying the root controller that some dragging is going on.
This will be used in order to prevent the controller from being hidden while the user is
dragging the `Seekbar`.

We can't forget about the **hardware keys**, if we don't listen for updates on those keys
we'll be out of sync. Don't worry, we got this covered too.

{% codeblock lang:java %}
@Override
protected void onAttachedToWindow() {
  super.onAttachedToWindow();
  getContext().registerReceiver(volumeReceiver,
    new IntentFilter("android.media.VOLUME_CHANGED_ACTION"));
}

@Override
protected void onDetachedFromWindow() {
  unregisterVolumeReceiver();
  getContext().unregisterReceiver(volumeReceiver);
}

private BroadcastReceiver volumeReceiver = new BroadcastReceiver() {
  @Override
  public void onReceive(Context context, Intent intent) {
      this.setProgress(audioManager.getStreamVolume(AudioManager.STREAM_MUSIC));
  }
};

{% endcodeblock %}

We can use a [BroadcastReceiver](http://developer.android.com/reference/android/content/BroadcastReceiver.html)
which alerts us when the volume has changed via hardware keys. We are going to make us of the view lifecycle
in order to register and unregister and update the `SeekBar` progress.

# Brightness SeekBar

Another interesting feature of any `MediaPlayer` is to adjust the screen brightness.
Currently, Android doesn't make it to easy for the user to do so and that's where [Fenster](https://github.com/malmstein/fenster) comes in.

The screen brightness is a [System Setting](http://developer.android.com/reference/android/provider/Settings.System.html#SCREEN_BRIGHTNESS)
that goes from 0 to 255. In order to change it, we just need to update the system setting with its new value.

Since we want to make it easy and modular, there is now a new class which will handle the
brightness update, the [BrightnessHelper](https://github.com/malmstein/fenster/blob/master/library/src/main/java/com/malmstein/fenster/helper/BrightnessHelper.java).
It just exposes a static method which needs a `Context` and the integer value with the new brightness.

{% codeblock lang:java %}
public static void setBrightness(Context context, int brightness){
  ContentResolver cResolver = context.getContentResolver();
  Settings.System.putInt(cResolver, Settings.System.SCREEN_BRIGHTNESS, brightness);
}
{% endcodeblock %}

Our approach is to use a `SeekBar`, similar as the `VolumeSeekBar` one. For that purpose, and in order make things more interactive
to the user, we'll use a `SeekBarChangeListener` to update the brightness meanwhile the `SeekBar` progress changes.

{% codeblock lang:java %}
public final OnSeekBarChangeListener brightnessSeekListener = new OnSeekBarChangeListener() {
  @Override
  public void onProgressChanged(SeekBar seekBar, int brightness, boolean fromUser) {
    setBrightness(brightness);
    setProgress(brightness);
  }

  @Override
  public void onStartTrackingTouch(SeekBar seekBar) {
    brightnessListener.onBrigthnessStartedDragging();
  }

  @Override
  public void onStopTrackingTouch(SeekBar seekBar) {
    brightnessListener.onBrightnessFinishedDragging();
  }
};
{% endcodeblock %}

## Skipping the video

The video skipping is pretty straight forward since the timeframe we are going to skip the video
forward or backwards is fixed. We just need to update the `SeekBar` progress:

{% codeblock lang:java %}
private void skipVideoForward() {
    int skipUnit = mProgress.getProgress() + SKIP_VIDEO_PROGRESS;
    mSeekListener.onProgressChanged(mProgress, skipUnit, true);
}
{% endcodeblock %}

# How the Controller updates it all

As a recap from the past blog post and this very same one, we are now able to detect gestures like
horizontal or vertical scrolls and horizontal or vertical flings. On the other hand, we also have
now two `Views` that can update the volume and the brightness so... let's put all that
together.

What we are going to do is to **update the video progress** when scroll or fling movements occur
and only **one finger** is being used to make the gesture. When **two or more fingers** are being used we
will **adjust the screen brightness or the volume**.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/multitouch.gif %}

How do we do that?

[MotionEvent](http://developer.android.com/reference/android/view/MotionEvent.html) already tells us
how many pointers are being used, that's basically telling us how many fingers are triggering the
event.

When a gesture with horizontal progress is detected we'll react on the video progress. One finger
for small updates and two fingers for skipping 30 seconds backwards or forwards. The delta parameter
tells us is the direction of the movement is right or left.

{% codeblock lang:java %}
@Override
public void onHorizontalScroll(MotionEvent event, float delta) {
  if (event.getPointerCount() == ONE_FINGER) {
      updateVideoProgressBar(delta);
  } else {
    if (delta > 0) {
        skipVideoForward();
    } else {
        skipVideoBackwards();
    }
  }
}
{% endcodeblock %}

When the movement is vertical, we'll update the volume and the brightness. One finger for the
volume and two or more for the brightness.

{% codeblock lang:java %}
@Override
public void onVerticalScroll(MotionEvent event, float delta) {
  if (event.getPointerCount() == ONE_FINGER) {
      updateVolumeProgressBar(delta);
  } else {
      updateBrightnessProgressBar(delta);
  }
}
{% endcodeblock %}

# Translating the gesture to the view

The approach is to translate the gestures on top the of `FensterGestureControllerView` into the
seekbars. Ideally, when making a gesture across the whole width or height of the `View` we'll
be also updating the `Seekbar` through it's whole range.

To do so, we need to use the movement delta and apply a scale to the `Seekbar`. Of course, we
also need to take into account the **max** value of the `Seekbar`.

{% codeblock lang:java %}
private int extractDeltaScale(int availableSpace, float deltaX, SeekBar seekbar) {
    int x = (int) deltaX;
    float scale;
    float progress = seekbar.getProgress();
    final int max = seekbar.getMax();

    if (x < 0) {
        scale = (float) (x) / (float) (max - availableSpace);
        progress = progress - (scale * progress);
    } else {
        scale = (float) (x) / (float) availableSpace;
        progress += scale * max;
    }

    return (int) progress;
}
{% endcodeblock %}

The **available space** corresponds to the size of the `FensterGestureControllerView`. The width
if the movement is horizontal or the height in case it's a vertical movement.

# Next steps

There is still much to do with `Fenster`. I'll be adding some ideas to the GitHub repo, please feel
free to [add your own requests](https://github.com/malmstein/fenster/issues) or even implement
them yourselves!
