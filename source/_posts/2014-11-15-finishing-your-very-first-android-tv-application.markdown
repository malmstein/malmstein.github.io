---
layout: post
title: "Finishing your very first Android TV Application"
date: 2014-11-15 12:46:54 +0000
comments: true
categories: androidtv lollipop
---

A few days ago [we started to build an Android TV application]("http://www.malmstein.com/blog/2014/10/21/building-applications-for-android-tv/") and the feedback was very good. Today, it's time to take another step since we'll be publishing our
[AndroidTV Explorer]("https://github.com/malmstein/AndroidTVExplorer") with some updates. After these updates, you'll be able to see the details of a video and stream it's content.

{% img https://raw.githubusercontent.com/malmstein/AndroidTVExplorer/master/art/detail.gif %}

<!-- more -->

Obviously, the application is not very useful right now. There is a list of videos and different
categories but there is no actual way to stream a video.

For that reason, there are two new components that we need to build. Those are
the video detail screen and the video player screen. Let's start with the detail one.

# DetailsFragment

To keep things simple for now and fulfill the purpose of the series, we will use the [Leanback Support Library](http://developer.android.com/tools/support-library/features.html#v17-leanback) again. This time, the `Widget` we are interested in is the [DetailsFragment](https://developer.android.com/reference/android/support/v17/leanback/app/DetailsFragment.html), which works in a similar way as the `BrowseFragment` used in the previous post.

{% img https://raw.githubusercontent.com/malmstein/AndroidTVExplorer/master/art/detail.png %}

## The Presenter

Let me introduce you to the [ClassPresenterSelector](https://developer.android.com/reference/android/support/v17/leanback/widget/ClassPresenterSelector.html). Following the same
pattern as the `Presenter` used at the `BrowseFragment`, the `DetailsFragment` is going to use a [DetailsOverviewRowPresenter](https://developer.android.com/reference/android/support/v17/leanback/widget/DetailsOverviewRowPresenter.html) which will present the information.

{% codeblock lang:java %}
ClassPresenterSelector ps = new ClassPresenterSelector();
ps.addClassPresenter(DetailsOverviewRow.class, dorPresenter);
ps.addClassPresenter(ListRow.class, new ListRowPresenter());
{% endcodeblock %}

As you can see, the `DetailsFragment` has two separate levels of information. There is a header or overview called `DetailsOverviewRowPresenter` which will show the detail of the video we interested in.

{% codeblock lang:java %}
DetailsOverviewRowPresenter dorPresenter = new DetailsOverviewRowPresenter(new VideoDetailsPresenter());
dorPresenter.setBackgroundColor(getResources().getColor(R.color.detail_background));
dorPresenter.setStyleLarge(true);
{% endcodeblock %}

That corresponds to the Video detail information. It's the [DetailsOverviewRow](https://developer.android.com/reference/android/support/v17/leanback/widget/DetailsOverviewRow.html) and consists of an image, a description view, and optionally a series of Actions that can be taken for the item.

{% codeblock lang:java %}
public class VideoDetailsPresenter extends AbstractDetailsDescriptionPresenter {

@Override
protected void onBindDescription(ViewHolder viewHolder, Object object) {
  Video details = (Video) object;

  viewHolder.getTitle().setText(details.getTitle());
  viewHolder.getSubtitle().setText(details.getStudio());
  viewHolder.getBody().setText(details.getDescription());
}
}
{% endcodeblock %}

The other important piece of information is the Related Videos section. In order to build that, the `DetailsFragment` asks the `ClassPresenterSelector` to show a `ListRowPresenter`. What are those related videos you may ask? We'll use the category of the video to get that information, basically showing all the videos from the same category.

{% codeblock lang:java %}
ArrayObjectAdapter adapter = new ArrayObjectAdapter(ps);
adapter.add(detailRow);

String subcategories[] = {getString(R.string.related_movies)};
HashMap<String, List<Video>> videos = VideoProvider.getMovieList();

ArrayObjectAdapter listRowAdapter = new ArrayObjectAdapter(new CardPresenter());
for (Map.Entry<String, List<Video>> entry : videos.entrySet()) {
  if (selectedVideo.getCategory().indexOf(entry.getKey()) >= 0) {
    List<Video> list = entry.getValue();
    for (int j = 0; j < list.size(); j++) {
      listRowAdapter.add(list.get(j));
    }
  }
}

HeaderItem header = new HeaderItem(0, subcategories[0], null);
adapter.add(new ListRow(header, listRowAdapter));

setAdapter(adapter);

{% endcodeblock %}

Good, we are now able to see the video detail and some related videos but... how do we interact with the video?

## Easily adding actions

For that purpose `DetailsFragment` exposes a very simple solution, the so called `Action`.

{% codeblock lang:java %}
DetailsOverviewRow detailRow = new DetailsOverviewRow(selectedMovie);
row.addAction(new Action(ACTION_WATCH_VIDEO, getResources().getString(
R.string.watch_video_1), getResources().getString(R.string.watch_video_2)));
{% endcodeblock %}

The element which will handle the `ActionClick` is the `DetailsOverviewRowPresenter`. Each of of the added actions is identified by it's `id`.

{% codeblock lang:java %}
dorPresenter.setOnActionClickedListener(new OnActionClickedListener() {
  @Override
  public void onActionClicked(Action action) {
    if (action.getId() == ACTION_WATCH_VIDEO) {
      Intent intent = new Intent(getActivity(), PlayerActivity.class);
      intent.putExtra(getResources().getString(R.string.video), selectedVideo);
      startActivity(intent);
    } else {
      Toast.makeText(getActivity(), action.toString(), Toast.LENGTH_SHORT).show();
    }
  }
});

{% endcodeblock %}

Right, so there is some interaction here, let's stream the video once and for all. The TV is the perfect form factor for displaying videos, so that should be pretty straight forward!

# PlayerFragment

Displaying a video is probably one of the most important use cases of Android TV. It's pretty clear at this point that the interaction between remote controller and device must be simple and
easy to use. For that reason the `TVPlayerFragment` was born.

The idea is pretty simple, tell the `Fragment` what to play and forget about the rest. One can press the keys in the remote and the expected behaviour will occur. The most interesting part is probably the direction arrows. When using them, I'd expect to seek over the video if pressing left or right. It's all about accessing the [KeyEvents](http://developer.android.com/reference/android/view/KeyEvent.html#KEYCODE_MEDIA_PLAY_PAUSE)

Since this is a `Fragment`, the `KeyEvents` come filtered by the `Activity`

{% codeblock lang:java %}

@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
  if (keyCode == KeyEvent.KEYCODE_DPAD_LEFT || keyCode == KeyEvent.KEYCODE_DPAD_RIGHT) {
    ((TVPlayerFragment) getFragmentManager().findFragmentById(R.id.player_fragment)).onKeyDown(keyCode);
  }
  return super.onKeyDown(keyCode, event);
}

...

public void onKeyDown(int keyCode) {
  int currentPos;

  int delta = (videoDuration / SCRUB_SEGMENT_DIVISOR);
  if (delta < MIN_SCRUB_TIME) {
    delta = MIN_SCRUB_TIME;
  }

  if (!areControllersVisible) {
    updateControllersVisibility(true);
  }

  switch (keyCode) {
    case KeyEvent.KEYCODE_DPAD_LEFT:
      currentPos = videoView.getCurrentPosition();
      currentPos -= delta;
      if (currentPos > 0) {
        play(currentPos);
      }
    case KeyEvent.KEYCODE_DPAD_RIGHT:
      currentPos = videoView.getCurrentPosition();
      currentPos += delta;
      if (currentPos < videoDuration) {
        play(currentPos);
      }
  }
}
{% endcodeblock %}

Even though we are displaying video in the TV we can't forget that this is still Android and we still have to deal with our beloved `MediaPlayer`. Some of you may have read about [Fenster](https://github.com/malmstein/fenster), a library I've been working on that simplifies the way we stream video content. A few weeks ago we explained how the `MediaPlayer` lifecycle works and [how to stream a video](http://www.malmstein.com/blog/2014/08/09/how-to-use-a-textureview-to-display-a-video-with-custom-media-player-controls/) in a very simple way.

Well, we still need to deal with those callbacks but we can do it a bit differently.

{% codeblock lang:java %}
private void setMediaPlayerCallbacks() {

  videoView.setOnErrorListener(new MediaPlayer.OnErrorListener() {

    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
      videoView.stopPlayback();
      mPlaybackState = PlaybackState.IDLE;
      controllersTimer = new Timer();
      controllersTimer.schedule(new BackToDetailTask(), CLOSE_VIDEO_TIME);
      return false;
    }
  });

  videoView.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {

    @Override
    public void onPrepared(MediaPlayer mp) {
      videoDuration = mp.getDuration();
      endText.setText(formatTimeSignature(videoDuration));
      videoSeekbar.setMax(videoDuration);
      restartSeekBarTimer();
    }
  });

  videoView.setOnCompletionListener(new MediaPlayer.OnCompletionListener() {

    @Override
    public void onCompletion(MediaPlayer mp) {
      stopSeekBarTimer();
      mPlaybackState = PlaybackState.IDLE;
      updatePlayButton(PlaybackState.IDLE);
      controllersTimer = new Timer();
      controllersTimer.schedule(new BackToDetailTask(), HIDE_CONTROLLER_TIME);
    }
  });
}
{% endcodeblock %}

We are basically saying whenever there's an error we'll just navigate back to the video detail screen. The other thing we are saying is that the `SeekBar` position will be updated every second and not in the main thread.

At this point, we can stream video content and enjoy it in our Android TV!

# Show me the code

You'll find the entire project on [Github](https://github.com/malmstein/AndroidTVExplorer) and I'll be adding
more features along with all the blog posts of this series. As always, all ~~trolling~~ feedback is more than welcome!
