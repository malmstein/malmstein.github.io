---
layout: post
title: "Your forms don't need to be ugly Part 3"
date: 2014-07-09 20:03:54 +0200
comments: true
published : true
categories: android forms floatinghints styling
---

This is the 3 part of a series of posts where we are trying to improve the ui and interaction with forms in Android. [Part 2](http://www.malmstein.com/blog/2014/06/15/your-forms-dont-need-to-be-ugly-part-2/)
introduced the [FloatingHints](http://www.github.com/malmstein/FloatingHints) library and showed how this simple pattern can make a big impact
in your forms.

The source code for Part 2 is [already available on Github](https://github.com/malmstein/UglyForms/tree/step_two).

On today's post, we will introduce an easy way of dealing with transitions between forms and how to respond to orientation changes

<!-- more -->

## The Form flow does not need to suck

{% img center https://raw.githubusercontent.com/malmstein/UglyForms/master/art/animate_form.gif %}

It's very common to see a welcome message when opening a form which is telling us a bit about the application or simply because someone decided that
we are all still living in the early days of the internet. Don't worry, we got you covered for that.

One of the most annoying things that may happen when navigation through a Login form is when pressing the **Register** button.
I've seen things, and I'm sure you also have! A common example is to jump to a whole new Activity, the user is going to see that ugly change of context,
the famous white background in the screen while the screens change or even worse... it  may as well send the user to the Browser just
because the application can't handle registrations.

Wouldn't it be nice if you could just have a nicer way of showing all the content?

## Animating transitions

There are several ways of animating views, `ViewPager`, `ViewAnimator`, `ViewSwitcher` being the most used examples.

Before start using a `ViewPager` and write adapters, fragments, etc... let's understand what we are talking about here. This is more a personal
point of view than anything else but, I'm just not a big fun of building a castle when I only need a hut. Our form currently has two main goals,
**Sign in** and **Register** users, that's 2 actions and bunch of `EditText`so let's keep it simple.

We are going to create three different views, one for **Register**, one for the **Welcome message** and the last one for **Signining in**. We will
then use an animator to make the transitions from one view to another. As discussed earlier, `ViewPager`could be a solution
but I don't see the need of using adapters and fragments for this purpose. `ViewSwitcher`only allows use to switch between two views so
the ideal candidate for this task is a `ViewAnimator`.

One thing though, we are not going to show one view after the other. Instead, we are going to switch between views with different animations
by sliding the form left and right depending on the action that the user wants to take. Right, how do we do that?

## The MultipleViewSwitcher

{% img center https://raw.githubusercontent.com/malmstein/UglyForms/step_three/art/animate_form.gif %}

As you can see, after showing the welcome message if the user wants to Register we'll slide the form from the left and if the user wants
to Log in we will slide the form from the right. To do that, the `MultipleViewSwitcher`needs to add
some new functionality to what the `ViewAnimator`can do.

The first addition is that we need to be able to change the initial page of the view.
There are several ways of doing this but since we are extending `ViewAnimator`and I really like to use the `ViewAttributes`we will
set this initial page as an attribute.

**attrs.xml**
{% codeblock lang:xml %}
  <declare-styleable name="MultipleViewSwitcher">
    <attr name="initialPage" format="integer" />
  </declare-styleable>

{% endcodeblock %}

Our new view is now able to use this attribute and change the displayed page once it's inflated. Note that this page change can't be done
before the view is inflated since it has not computed yet all all it's children nor added them to the `ViewGroup`.

{% codeblock lang:java %}

  public class MultipleViewSwitcher extends ViewAnimator {

      private int initialChild;

      public MultipleViewSwitcher(Context context) {
          this(context, null);
      }

      public MultipleViewSwitcher(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }

      public MultipleViewSwitcher(Context context, AttributeSet attrs, int defStyle) {
          super(context, attrs);

          TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.MultipleViewSwitcher, defStyle, 0);
          initialChild = a.getInt(R.styleable.MultipleViewSwitcher_initialPage, 0);

          a.recycle();
      }

      @Override
      protected void onFinishInflate() {
          super.onFinishInflate();
          displayInitialViewIfNecessary();
      }

      private void displayInitialViewIfNecessary() {
          if (initialChild > 0){
              setDisplayedChild(initialChild);
          }
      }

    }

{% endcodeblock %}

As you all know, the first view in the `ViewAnimator`has the id 0 so we are going to set our initialPage as 1:

{% codeblock lang:xml %}
    <com.malmstein.widgets.MultipleViewSwitcher
      android:id="@+id/form_animator"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      app:initialPage="1">

      <include
        android:id="@+id/form_register"
        layout="@layout/view_form_register" />

      <include
        android:id="@+id/form_welcome_message"
        layout="@layout/view_form_welcome" />

      <include
        android:id="@+id/form_login"
        layout="@layout/view_form_login" />

    </com.malmstein.widgets.MultipleViewSwitcher>

{% endcodeblock %}

Cool, now all the form is in place and our first page is the **Welcome Message**. Earlier we said that we wanted to animate the
transitions between forms, and that's exactly what the `MultipleViewSwitcher`is made for. Basically, there are two methods and attributes
that we are going to use in order to achieve this `setInAnimation(Animation inAnimation)`and `setOutAnimation(Animation inAnimation)`

As you've seen, the forms are not always sliding into the same direction but changing based on the current page. In order to achieve that we are also
going to add some changes to the transition logic so this can be easily handled.

{% codeblock lang:java %}
    /**
     * Animates the view to the next children using the animations specified
     *
     * @param inAnimation The inAnimation
     * @param outAnimation The outAnimation
     */
    public void animateNext(Animation inAnimation, Animation outAnimation) {
        setInAnimation(inAnimation);
        setOutAnimation(outAnimation);
        showNext();
    }

    /**
     * Animates the view to the next children using the animations specified
     *
     * @param context              The application's environment.
     * @param inAnimationResource  The resource id of the inAnimation.
     * @param outAnimationResource The resource id of the outAnimation.
     */
    public void animateNext(Context context, int inAnimationResource, int outAnimationResource) {
        animateNext(AnimationUtils.loadAnimation(context, inAnimationResource),
                AnimationUtils.loadAnimation(context, outAnimationResource));
    }

{% endcodeblock %}

The view allows us to set two Animations which we will use to animate the views accordingly. Looking into
the [ViewAnimator](http://developer.android.com/reference/android/widget/ViewAnimator.html) implementation one can
see that it's holding refference to both the **In** and the **Out** Animations, so there is no need for our new View to add more overhead to it.
It is our responsability though to keep an eye on how many Animations we are loading and how we do it. Same as the `ViewAnimator`, the `MultipleViewSwitcher`
can receive a new Animation by parameter or the resourceId of it which we will then load with `AnimationUtils.loadAnimation(context, animationResource)`

At this point we hace achieved all the goals of the wanted but there is a small annoying issue, when rotating the device we are not saving the
current displayed child `ViewAnimator`does not do it either, so it would be the Activity/Fragment responsibility. I'd much rather have this solved
in the View so we are going to save the state of the View when rotation.

## Saving the state of the view

Since our form is now composed by more than one view, it would be nice to save the displayed child on orientation changes. No one wants to move the device from portrait to landscape
and realise that the form is now showing the initial page. This view is not just usable in forms, one can easily use it to handle the on boarding wizard of an application.
Can you imagine how frustrated the user would be if after swiping through four screens, the device is rotated and the wizards needs to be completed from the beginning?

Android uses `Parcelable`to save the [instance state](http://developer.android.com/reference/android/view/View.html#onSaveInstanceState()). I've seen plenty of implementations
creating their own Parcelable, etc... but the easiest way in my opinion is using a [Bundle](http://developer.android.com/reference/android/os/Bundle.html) (which implements Parcelable)

{% codeblock lang:java %}

    @Override
    public Parcelable onSaveInstanceState() {
        Bundle bundle = new Bundle();
        bundle.putParcelable("instanceState", super.onSaveInstanceState());
        bundle.putInt("displayedChild", getDisplayedChild());
        return bundle;
    }

    @Override
    public void onRestoreInstanceState(Parcelable state) {
        if (state instanceof Bundle) {
            Bundle bundle = (Bundle) state;
            initialChild = bundle.getInt("displayedChild");
            setDisplayedChild(initialChild);
            state = bundle.getParcelable("instanceState");
        }
        super.onRestoreInstanceState(state);
    }


{% endcodeblock %}

When the device's orientation changes, the view itself will save it's instance state and restore the displayed child. It's very important that the View has it's own id defined
in the XML layout, if it doesn't the `onSaveInstanceState()`and `onRestoreInstanceState()`won't be called and state won't be restored.

## Show me the code

If this caught your attention, please check the source code for Part 3 [already available on Github](https://github.com/malmstein/UglyForms/tree/step_three).

The next part will be last one and will add some love to the overall look and feel in addition to some interesting ideas about buttons and input keyboard handling.
