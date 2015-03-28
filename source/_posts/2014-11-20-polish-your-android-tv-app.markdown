---
layout: post
title: "Polish your Android TV App"
date: 2014-11-20 10:31:47 -0800
comments: true
published: false
categories: androidtv, leanback, androidtvexplorer
---



<!-- more -->

# Add your videos to the recommendation service

Served as notifications to Android TV

Recommendation Receiver

  Updates every half an hour
  Bootcompleted permission

Recommendation Service

  Builds the notification and the PendingIntent
  Ensure a unique PendingIntents, otherwise all recommendations end up with the same
  PendingIntent



# ControlButtonPresenterSelector

# PlayOverlayFragment

# Support both mobile and tv app

if (autoPlay) {
  if (mIsOnTv) {
    // This will update the player controls and the activity will receive the callback
    // OnPlayPauseClickedListener.onFragmentPlayPause(Video, int, Boolean)
    mPlaybackOverlayFragment.pressPlay();
    } else {
      player.setPlayWhenReady(true);
    }
    autoPlay = false;
  }
