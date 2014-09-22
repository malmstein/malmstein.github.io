---
layout: post
title: "How to use a TextureView to display a video with custom media player controls"
date: 2014-08-09 20:00:00 +0100
comments: true
categories: video textureview mediaplayer fenster novoda
---

For the past months at [Novoda](http://www.novoda.com) I've been working on several video applications [ARTE](https://play.google.com/store/apps/details?id=tv.arte.plus7),
[Digital Concert Hall](https://play.google.com/store/apps/details?id=com.novoda.dch) and [MUBI](https://play.google.com/store/apps/details?id=com.mubi).

As you might figure by now, the common feature of these applications is to display a video with customised media player controls. I thought it would be
a good time to share some knowledge and show some ideas around video playing on Android. A couple of months ago I a [gave a talk](https://speakerdeck.com/malmstein/streaming-the-droid)
 about it at **Droidcon Berlin** showing some code but I never had the proper amount of time to make it into a library.

<!-- more -->

## Media Player lifecycle

The first thing we need to understand before playing videos or audio is the lifecycle of the [Media Player](http://developer.android.com/reference/android/media/MediaPlayer.html)

{% img http://developer.android.com/images/mediaplayer_state_diagram.gif %}

As you see it's pretty straight forward, **there are only 10 different states**...

The most important piece of information that can be extracted from this diagram is that one needs to be very strict regarding the
behaviour of the `MediaPlayer`. It's very easy to follow the steps in the wrong order which will lead to frustration since debug information that the
`MediaPlayer` provides is pretty bad:

{% codeblock lang:java %}
private static int getErrorMessage(final int frameworkError) {
    int messageId = R.string.play_error_message;

    if (frameworkError == MediaPlayer.MEDIA_ERROR_IO) {
        Log.e(TAG, "TextureVideoView error. File or network related operation errors.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_MALFORMED) {
        Log.e(TAG, "TextureVideoView error. Bitstream is not conforming to the related coding standard or file spec.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_SERVER_DIED) {
        Log.e(TAG, "TextureVideoView error. Media server died. In this case, the application must release the MediaPlayer object and instantiate a new one.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_TIMED_OUT) {
        Log.e(TAG, "TextureVideoView error. Some operation takes too long to complete, usually more than 3-5 seconds.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_UNKNOWN) {
        Log.e(TAG, "TextureVideoView error. Unspecified media player error.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_UNSUPPORTED) {
        Log.e(TAG, "TextureVideoView error. Bitstream is conforming to the related coding standard or file spec, but the media framework does not support the feature.");
    } else if (frameworkError == MediaPlayer.MEDIA_ERROR_NOT_VALID_FOR_PROGRESSIVE_PLAYBACK) {
        Log.e(TAG, "TextureVideoView error. The video is streamed and its container is not valid for progressive playback i.e the video's index (e.g moov atom) is not at the start of the file.");
        messageId = R.string.play_progressive_error_message;
    }
    return messageId;
}

{% endcodeblock %}

This information can be accessed using two different listeners from the `MediaPlayer`, [onInfoListener](http://developer.android.com/reference/android/media/MediaPlayer.OnInfoListener.html)
and [onErrorListener](http://developer.android.com/reference/android/media/MediaPlayer.OnErrorListener.html). We are learning all this before actually displaying
any content, and you thought this was going to be easy!

Now we understand how the `MediaPlayer` works, the next step is to decide how are we going to display the video, what `View` are we going to use?

## TextureView vs SurfaceView

The easiest way of displaying a video in Android is using a [VideoView](http://developer.android.com/reference/android/widget/VideoView.html). With very few
lines of code you'll be able to display a video from a local source or from the internet and it will just play. **This is a very good approach if you
don't need any fancy stuff**, no customisation or no callbacks. In our case, customisation was needed hence callbacks were needed and this solution was simply
just not good enough.

In the first place my thought was to extend `VideoView` and add some functionality but if you've checked the [source code](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/widget/VideoView.java),
`VideoView` is extending `SurfaceView` and that has some limitations and constraints. `SurfaceView` provides a dedicated drawing surface embedded inside of a view hierarchy,
the layout of which can be defined, in size and format. It also takes care of placing the surface at the correct location on the screen.

The main limitation is that **SurfaceView can't be animated, moved or transformed**, that's because it's created in a separate window and not
treated as a `View`. That was a big problem considering what we wanted to achieve so we turned our eyes
towards [TextureView](http://developer.android.com/reference/android/view/TextureView.html).

It's not all bad news for `SurfaceView`, **Chrome** used to use `TextureView` as the compositor target surface but they went back to using
`SurfaceView`. These are the main reasons:

* Because of its invalidation and buffering behaviour, TextureView adds 1-3 extra frames of latency to display updates.
* TextureView is always composited using GL, whereas SurfaceTexture can be backed by a hardware overlay which uses less memory bandwidth and power.
* The internal buffer queue of TextureView can end up using more memory than a SurfaceView.
* We weren't really doing anything useful with the animation and transform capabilities of TextureView.

Full discussion in Google Groups can be found [here](https://groups.google.com/a/chromium.org/forum/#!topic/graphics-dev/Z0yE-PWQXc4)

We wanted to transform and animate that view so our choice was already made.

## Custom Controls

So the video is going to be displayed in a `TextureView`, now we need a way of interacting with the video. Yes, the media player controls.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/custom_controls.png %}

As you can see we decided to use a similar approach to what **Youtube** is doing, the custom controller occupies the whole screen making it very easy to use.
Quite frankly this is much more appealing from my point of view. We've all seen these designs for video players where the controller is aligned to the
bottom of the screen (inspired in certain other platform) and I certainly believe this approach makes more sense.

{% codeblock lang:xml %}
<com.malmstein.fenster.VideoTouchRoot
  xmlns:android="http://schemas.android.com/apk/res/android"
  android:id="@+id/media_controller_touch_root"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:background="@color/default_bg">

....

</com.malmstein.fenster.VideoTouchRoot>

{% endcodeblock %}

The most important part of this custom controller is **the root layout which is going to enable / disable the controller itself**. That is the `VideoTouchRoot` and
it's nothing else than a `FrameLayout` we put as a layout root to get notified about any screen touches. The `PlayerController` will set a listener and show or
hide the view accordingly

{% codeblock lang:java %}
VideoTouchRoot touchRoot = (VideoTouchRoot) v.findViewById(R.id.media_controller_touch_root);
touchRoot.setOnTouchReceiver(this);
{% endcodeblock %}

**How does the PlayerController work?**

Basically, the big part of the job is done when the `PlayerController` needs to be attached. It's the `TextureView` who takes care of this job, setting an anchor
view which is the `PlayerController` itself.

Once this view is anchored it's just a matter of setting listeners to all the events we want to track (**Play**/**Pause**/**Seek**) and handle the communication
between the view and the `MediaPlayer`.

{% codeblock lang:java %}
@Override
public void setAnchorView(final View view) {
    mAnchor = view;
    LayoutParams frameParams = new LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT
    );

    removeAllViews();
    View v = makeControllerView();
    addView(v, frameParams);
}
{% endcodeblock %}

## TextureVideoView

At this point we know how the controller is shown / hidden and which component is that view anchored to, but we are missing the big part of this puzzle
which is the View who handles it all, the `TextureVideoView`.

This view is the only one communicating with the `MediaPlayer`, which allows us to handle on the interaction between the controller and the player
transparently to the other parts of the application. Going back to the initial state diagram, one can see that it would be very helpful to save
the actual state of the player in order to know which steps can be followed. The `TextureVideoView` **keeps track internally of that state**.

Another important aspect of the view consists in handling all the listeners that the `MediaPlayer` exposes

{% codeblock lang:java %}
mMediaPlayer.setOnPreparedListener(mPreparedListener);
mMediaPlayer.setOnVideoSizeChangedListener(mSizeChangedListener);
mMediaPlayer.setOnCompletionListener(mCompletionListener);
mMediaPlayer.setOnErrorListener(mErrorListener);
mMediaPlayer.setOnInfoListener(mInfoListener);
mMediaPlayer.setOnBufferingUpdateListener(mBufferingUpdateListener);

mMediaPlayer.setDataSource(getContext(), mUri, mHeaders);
mMediaPlayer.setSurface(new Surface(mSurfaceTexture));
mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mMediaPlayer.setScreenOnWhilePlaying(true);
mMediaPlayer.prepareAsync();

mCurrentState = STATE_PREPARING;
{% endcodeblock %}

As you can see, there is a big chunk of listeners to be set here, not the nicest looking part but the most helpful of all. Not only we've set
all the listeners but also created the [SurfaceTexture](http://developer.android.com/reference/android/graphics/SurfaceTexture.html) **where the video
will be displayed**, set the DataSource with the video information and last but not least important, asked the `MediaPlayer` to prepare itself in order
to start playing the video.

Important here not to call `prepare()` but use `prepareAsync()` instead. As you can imagine the `prepare()` method is **run in the UI thread** which will block
the application, by using the `onPreparedListener` we'll be notified when the `MediaPlayer` can start playing the video. It's just another listener and
we love them, don't we?

**Notify when the video is playing**

From all those listeners that we've set earlier, perhaps the one which tells us more about the `MediaPlayer` status is the `onInfoListener()`:

{% codeblock lang:java %}
private final OnInfoListener onInfoToPlayStateListener = new OnInfoListener() {

  @Override
  public boolean onInfo(final MediaPlayer mp, final int what, final int extra) {
      if (noPlayStateListener()) {
          return false;
      }

      if (MediaPlayer.MEDIA_INFO_VIDEO_RENDERING_START == what) {
          onPlayStateListener.onFirstVideoFrameRendered();
          onPlayStateListener.onPlay();
      }
      if (MediaPlayer.MEDIA_INFO_BUFFERING_START == what) {
          onPlayStateListener.onBuffer();
      }
      if (MediaPlayer.MEDIA_INFO_BUFFERING_END == what) {
          onPlayStateListener.onPlay();
      }

      return false;
  }
};
{% endcodeblock %}

These three flags give us vital information, they will tell us **when the first frame of the video starts to play**, when the **MediaPlayer starts buffering** and
when the **video is playing again after buffering**. Finally all this information will be sent to the PlayerController in order to show / hide the controls
and the loading view.

## Introducing Fenster

To wrap things up, it's always better to have a project to look into it. That's why I've created [Fenster](https://github.com/malmstein/fenster), a library
where you'll find all these views we've been talking about in this post and a nice **demo project** to show it all.

{% img https://raw.githubusercontent.com/malmstein/Fenster/master/art/video_example.gif %}

As always, **fork it**, **use it**, and if you want to contribute [pull requests](https://github.com/malmstein/fenster/pulls) are more than welcome!
