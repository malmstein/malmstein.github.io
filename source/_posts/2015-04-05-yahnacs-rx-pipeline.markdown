---
layout: post
title: "Building Yahnac's Rx pipeline"
date: 2015-04-05 09:24:49 +0100
comments: true
categories: yahnac hackernews contentprovider rxjava firebase
---

Following the [introduction post about Yahnac](http://www.malmstein.com/blog/2015/03/28/introducing-yahnac-where-rxjava-meets-content-providers/) I thought it'd be a good idea to share the learnings from [Rx Java](http://reactivex.io/) and how I've used it on [Yahnac](https://github.com/malmstein/yahnac).

If you are interested in **how to build the Rx pipeline alongside Firebase integration** and how to make it work with [Content Providers](http://developer.android.com/guide/topics/providers/content-providers.html), please Read on!

{% img http://www.jsg.utexas.edu/lacp/files/PGC_Pipeline_Data.jpg %}

<!-- more -->

# What is the Rx pipeline in Yahnac?

Basically, it's everything that happens between someone subscribes to an `Observable` in the networking layer until the data is stored in the `Content Provider`. For those familiar with [RxJava](https://github.com/ReactiveX/RxJava) this will sound a bit of a simple explanation. There's people why smarter than me that have explained [the principles of Rx Java on Android](https://speakerdeck.com/mttkay/conquering-concurrency-bringing-the-reactive-extensions-to-the-android-platform) and [how to handle data-flows the reactive way](http://www.slideshare.net/rolandkuhn/reactive-streams).

# Retrieving data from Firebase

Before getting into detail of how the pipeline works in our case, we first need to understand how to retrieve the data. In order to do so, `Firebase` allows us to read two different event types **Value** and **Child Added**, [both events work differently](https://www.firebase.com/docs/android/guide/retrieving-data.html#section-types).  

## Value event

The event callback is passed a snapshot containing all data at that location, in this example we are retrieving a list of blog posts

{% codeblock lang:java %}
Firebase ref = new Firebase("https://docs-examples.firebaseio.com/web/saving-data/fireblog/posts");
ref.addValueEventListener(new ValueEventListener() {
  @Override
  public void onDataChange(DataSnapshot snapshot) {
    System.out.println(snapshot.getValue());
  }

  @Override
  public void onCancelled(FirebaseError firebaseError) {
    System.out.println("Error: " + firebaseError.getMessage());
  }
});
{% endcodeblock %}

## Child Added event

Unlike the value event which returns the entire contents of the location, the `onChildAdded` event is triggered once for each existing child and then again every time a new child is added to the specified path.

{% codeblock lang:java %}
Firebase ref = new Firebase("https://docs-examples.firebaseio.com/web/saving-data/fireblog/posts");
ref.addChildEventListener(new ChildEventListener() {
  @Override
  public void onChildAdded(DataSnapshot snapshot, String previousChildKey) {
    Map<String, Object> newPost = (Map<String, Object>) snapshot.getValue();
    System.out.println("Author: " + newPost.get("author"));
    System.out.println("Title: " + newPost.get("title"));
  }
});
{% endcodeblock %}

The object retrieved `DataSnapshot` allows us to read the data using the `getValue()` method and will return  `Boolean`, `Long`, `Double`, `Map<String, Object>` or `List<Object>`. If no data exists at the location, the snapshot will return null.

# Retrieving the Top Stories

The [Hacker News API](https://github.com/HackerNews/API) exposes a reference for each one of the different categories available, for the purpose of this example we are going to focus on the **Top Stories** one.

The available `Firebase` reference will send a [list which contains the id fields of the stories](https://hacker-news.firebaseio.com/v0/topstories). However, this does not give us all the information we need. For that matter, there is an endpoint which will retrieve [data from a specific story](https://hacker-news.firebaseio.com/v0/item/8744658).

{% codeblock lang:java %}
Firebase topStories = new Firebase("https://hacker-news.firebaseio.com/v0/topstories");
topStories.addValueEventListener(new ValueEventListener() {
    @Override
    public void onDataChange(DataSnapshot dataSnapshot) {
        //do something with this
    }

    @Override
    public void onCancelled(FirebaseError firebaseError) {
        Log.d(firebaseError.getCode());
    }
});
{% endcodeblock %}

The `DataSnapshot` contains a `List` of integers, which correspond to each one of the items in the **Top Stories** page, ordered by rank. At this point we'll need to ask for the data related to each story.

{% codeblock lang:java %}
final Firebase story = new Firebase("https://hacker-news.firebaseio.com/v0/item/" + id);
  story.addValueEventListener(new ValueEventListener() {
      @Override
      public void onDataChange(DataSnapshot dataSnapshot) {
          Map<String, Object> newItem = (Map<String, Object>) dataSnapshot.getValue();
          //do something with this
      }

      @Override
      public void onCancelled(FirebaseError firebaseError) {
          Log.d(firebaseError.getCode());
      }
  });
{% endcodeblock %}

Since every story item has several fields, `Firebase` will send a `Map` with the data. Note that here we are asking for the **Value** events and not the **Child Event**. There is a reason for that which directly depends on the **Rx pipeline** we were talking earlier.

Let's take a step back and take a look at the bigger picture. The main goal of this networking layer is to retrieve the data and serve it to the database. Since we are using `Content Provider` the data type that we can store is `ContentValue`. Therefore, the ouptut of the **Rx pipeline** should be a list of `ContentValues` which we'll then insert into the database.

# Building the pipeline

So, how do we generate those `ContentValues` if all we have is a list of story ids? Well, we'll need to use the concept of [Operator](http://reactivex.io/documentation/operators.html). **Rx** allows use to make operations between different `Observables` in order to get the desired output. If we follow the **decision tree of Observable Operators**, we get to the conclusion that we need to use [Flat Map](http://reactivex.io/documentation/operators/flatmap.html)

{% img http://reactivex.io/documentation/operators/images/flatMap.c.png %}

`Flat Map` transform the items emitted by an `Observable` into `Observables`, then flatten the emissions from those into a single `Observable`. In our case that very last `Observable` will contain all the `ContentValues` that will be inserted into the `ContentProvider`. Let's see how do we do that.

As we've seen earlier, **the first thing we can get from the API is a list of ids** corresponding to the stories. After that, we'll need to get the data for every single one of those ids. Once we get the data from a story item, we'll need to create the `ContentValues` and finally inserting them into the `ContentProvider`.

{% codeblock lang:java %}
public Observable<List<ContentValues>> getStories(final Story.TYPE type) {
  return Observable.create(new Observable.OnSubscribe<DataSnapshot>() {

    @Override
    public void call(final Subscriber<? super DataSnapshot> subscriber) {
      Firebase topStories = getStoryFirebase(type);
      topStories.addValueEventListener(new ValueEventListener() {

        @Override
        public void onDataChange(DataSnapshot dataSnapshot) {
          subscriber.onNext(dataSnapshot);
          subscriber.onCompleted();
        }

        @Override
        public void onCancelled(FirebaseError firebaseError) {
          Log.d(firebaseError.getCode());
        }
    });
  }
{% endcodeblock %}

As you can see, the method `getStories()` is the public method exposed which builds the pipeline. Using the `Flat Map` operator, the `DataSnapshot` will be sent to the next `Observable` which will handle the next operation. You might ask, why don't you use the **Child Added** event that `Firebase` offers? Well, the reason is because using **Child Added** event the `Subscriber` would never complete. We would get a `Callback` for every single new item and we'd be able to send to the next `Observable`, but the pipeline would be broken since the operations on that `Observable` would only start once `onCompleted()` is called in the previous `Observable`.

{% codeblock lang:java %}
.flatMap(new Func1<DataSnapshot, Observable<Pair<Integer, Long>>>() {
  @Override
  public Observable<Pair<Integer, Long>> call(final DataSnapshot dataSnapshot) {

    return Observable.create(new Observable.OnSubscribe<Pair<Integer, Long>>() {
      @Override
      public void call(Subscriber<? super Pair<Integer, Long>> subscriber) {
        for (int i = 0; i < dataSnapshot.getChildrenCount(); i++) {
          Long id = (Long) dataSnapshot.child(String.valueOf(i)).getValue();
          Integer rank = Integer.valueOf(dataSnapshot.child(String.valueOf(i)).getKey());
          Pair<Integer, Long> storyRoot = new Pair<>(rank, id);
          subscriber.onNext(storyRoot);
        }
        subscriber.onCompleted();
      }
  });
}
{% endcodeblock %}

First `FlatMap` here, we are receiving a list of ids and sending each one of them to the next `Observable`. Lucky enough, `DataSnapshot` allows us to access the items using `dataSnapshot.child()` and `dataSnapshot.getChildrenCount()`. It's just not enough with sending the **id**, we also want to send the **rank** of the story item since we want to be sure that the order is correct. For that reason we use the handy [Pair](http://developer.android.com/reference/android/util/Pair.html) tuple object.

{% codeblock lang:java %}
}).flatMap(new Func1<Pair<Integer, Long>, Observable<ContentValues>>() {
  @Override
  public Observable<ContentValues> call(final Pair<Integer, Long> storyRoot) {
    return Observable.create(new Observable.OnSubscribe<ContentValues>() {
      @Override
      public void call(final Subscriber<? super ContentValues> subscriber) {
        final Firebase story = new Firebase("https://hacker-news.firebaseio.com/v0/item/" + storyRoot.second);
        story.addValueEventListener(new ValueEventListener() {
          @Override
          public void onDataChange(DataSnapshot dataSnapshot) {
            Map<String, Object> newItem = (Map<String, Object>) dataSnapshot.getValue();
            subscriber.onNext(mapStory(newItem, type, storyRoot.first));
            subscriber.onCompleted();
          }

          @Override
          public void onCancelled(FirebaseError firebaseError) {
            Log.d(firebaseError.getCode());
          }
        });
      }
    });
  }
}).toList();
{% endcodeblock %}

What makes all the magic here is `toList()`. The operator [to](http://reactivex.io/documentation/operators/to.html) will convert the `Observable` into another object, in this case a `List`.

{% img http://reactivex.io/documentation/operators/images/to.c.png %}

Once we've got to this point, the `Observable` will provide a `List<ContentValues>` which is exactly what we wanted. Now we just need to know who is going to take care of the data, insert it in the `ContentProvider` and notify to the `Subscriber` that all the job is done.

# Data repository

With this purpose in mind, and with the aim of having a single entry point for the networking layer, the `DataRepository` was created, which will take care of the so called **pipeline**.

{% codeblock lang:java %}
public Observable<Integer> getStories(final Story.TYPE type) {
  return api.getStories(type)
    .flatMap(new Func1<List<ContentValues>, Observable<Integer>>() {
      @Override
      public Observable<Integer> call(final List<ContentValues> stories) {
        return Observable.create(new Observable.OnSubscribe<Integer>() {
          @Override
          public void call(Subscriber<? super Integer> subscriber) {
            dataPersister.persistStories(stories);
            subscriber.onNext(stories.size());
            subscriber.onCompleted();
          }
        });
      }
  });
}
{% endcodeblock %}

For now we are just notifying the `Subscriber` with the amount of stories inserted, but actually, that could be removed with no impact. There is just one more thing, how is all this data inserted in the `ContentProvider`?

## Inserting the data

One of the features of the `ContentProvider` is to insert data in bulks. Yes, you got it, an array of `ContentValues` is exactly what's needed.

{% codeblock lang:java %}
public int persistStories(List<ContentValues> topStories) {
  ContentValues[] cvArray = new ContentValues[topStories.size()];
  topStories.toArray(cvArray);

  return contentResolver.bulkInsert(URI, cvArray);
}
{% endcodeblock %}

With this simple action, we've told the `ContentResolver` to insert all the data from the **Top Stories** page.

{% codeblock lang:java %}
@Override
public int bulkInsert(Uri uri, ContentValues[] values) {
    final SQLiteDatabase db = mOpenHelper.getWritableDatabase();
    ...
    db.beginTransaction();
    try {
        for (ContentValues value : values) {
            Cursor exists = db.query(...);

            if (exists.moveToLast()) {
                long _id = db.update(...);
            } else {
                long _id = db.insert(...);
            }
            exists.close();
        }
        db.setTransactionSuccessful();
    } finally {
        db.endTransaction();
    }
    getContext().getContentResolver().notifyChange(URI, null);
    return returnCount;
{% endcodeblock %}

There is a reason why we are using `bulkInsert` and not `insert`. If we were to use `insert`, the `ContentResolver` will notify all it's `Oberservers` every time an item is inserted. **Every single call to Top Stories could return over 500 stories, that results in 500 x 8 ContentValues**. Meaning the `Loader` listening to this specific `URI` would be notified 4000 times, which would make the `Adapter` to be refreshed 4000 times too.

By calling `bulkInsert` we are only notifying the `Loader` once, that's a huge improvement don't you think. **If you are inserting more than one `ContentValues` into the `ContentResolver`, make sure you are always using `bulkInsert`**.

It's always better to see the code, right? [The source is available on Github](https://github.com/malmstein/yahnac).

In the next post we'll talk about the UI, and the lessons learned with `RecyclerView`, `Loaders` and `AppCompat`.
