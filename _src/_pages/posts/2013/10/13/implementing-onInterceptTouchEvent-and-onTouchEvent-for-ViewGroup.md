---
title: Correctly implementing onInterceptTouchEvent and onTouchEvent methods for ViewGroup
layout: post
format: markdown
type: post
tags: android, viewgroup, touch-events
slug: implementing-onInterceptTouchEvent-and-onTouchEvent-for-ViewGroup.html
timestamp: 1381596101
---

There's already an [article](http://developer.android.com/training/gestures/viewgroup.html) on this topic on the Android developer website, I write this post just to elaborate further on the topic to make it easier to understand.

`onTouchEvent` is declared in the `View` class, because any kind of views can handle touch events. While `onInterceptTouchEvent` is declared in the `ViewGroup` class, because `ViewGroup` is a *view container*, and only *view containers* may need to intercept touch events, like `ScrollView`, which intercepts *scroll* events to scroll its child views. When handling touch events in a `ViewGroup`, you will most probably need to implement both of these methods.

`onInterceptTouchEvent` is always the entry point for the first touch event, i.e. the `ACTION_DOWN` event, and in this method, you return `true` to indicate that the current `ViewGroup`(or more precisely, current `ViewGroup`'s `onTouchEvent` method) wants to handle all the touch events till next `ACTION_UP` or `ACTION_CANCEL`, and in most cases, the touch events between `ACTION_DOWN` and `ACTION_UP` or `ACTION_CANCEL` are `ACTION_MOVE`s, which will normally be recognized as scrolling/fling gestures.

Let's say we have the following layout(MyViewGroup extends `ViewGroup`):

    <MyViewGroup
            android:layout_width="fill_parent"
            android:layout_height="fill_parent">
        <Button
                android:id="@+id/btnTest"
                android:layout_width="fill_parent"
                android:layout_height="wrap_content"
                android:text="Test" />

        ...many other views here

    </MyViewGroup>

I want to implement vertical scrolling for `MyViewGroup`, two cases need to be taken into account: 

1. I start scrolling by putting my finger on `MyViewGroup` itself and move vertically my finger on the screen.
2. I start scrolling by putting my finger on a child view(*btnTest* button in the layout above) of `MyViewGroup` and move vertically my finger on the screen.

To address these cases, we need to implement both `onInterceptTouchEvent` and `onTouchEvent`. 

For case one, things are easy, no matter what we return from `onInterceptTouchEvent`, the touch events will be propagated to `MyViewGroup`'s `onTouchEvent` method, the reason is that the target of the `ACTION_DOWN` event is `MyViewGroup` itself. `onTouchEvent` should always return **true** in this case, and it is supposed to handle all touch events that follow. Once `MyViewGroup`'s `onTouchEvent` returns true, `onInterceptTouchEvent` will **NOT** receive any touch events afterwards.

For case two, `MyViewGroup`'s `onTouchEvent` method will **NOT** receive an `ACTION_DOWN` event, `onInterceptTouchEvent` will receive the the `ACTION_DOWN` event, but it must **NOT** return true for this event, because it is not the target of the event in this case and we have no idea whether the user wants to simply *tap* the button or *scroll* the content of `MyViewGroup`. When `onInterceptTouchEvent` starts receiving `ACTION_MOVE` events, we assume the user **might** want to scroll, and save the distance that the user's finger travels on the Y-axis, when the distance exceeds a preset threshold(say 40dp), we can be sure that the user wants to *scroll*, so we return *true* immediately. Once we return true from `onInterceptTouchEvent`, the button will receive its **last** touch event - `ACTION_CANCEL`, and all the touch events that follow will be directly propagated to `MyViewGroup`'s `onTouchEvent` method and be handled the same way as in case one.

Note, for the threshold to determine a scroll gesture, we can use `ViewConfiguration.get(context).getScaledTouchSlop()`, touch slop is that threshold.

Let's make a conclusion: the `onInterceptTouchEvent` method, as its name implies, has the power of intercepting any touch events and deciding who(itself or its child views) handles the events.

To make this post complete, some code is attached.

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mLastX = event.getX();
                mLastY = event.getY();
                mStartY = mLastY;
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mIsBeingDragged = false;
                break;
            case MotionEvent.ACTION_MOVE:
                float x = event.getX();
                float y = event.getY();
                float xDelta = Math.abs(x - mLastX);
                float yDelta = Math.abs(y - mLastY);

                float yDeltaTotal = y - mStartY;
                if (yDelta > xDelta && Math.abs(yDeltaTotal) > mTouchSlop) {
                    mIsBeingDragged = true;
                    mStartY = y;
                    return true;
                }
                break;
        }

        return false;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                mIsBeingDragged = false;
                break;
            case MotionEvent.ACTION_MOVE:
                float x = event.getX();
                float y = event.getY();

                float xDelta = Math.abs(x - mLastX);
                float yDelta = Math.abs(y - mLastY);

                float yDeltaTotal = y - mStartY;
                if (!mIsBeingDragged && yDelta > xDelta && Math.abs(yDeltaTotal) > mTouchSlop) {
                    mIsBeingDragged = true;
                    mStartY = y;
                    yDeltaTotal = 0;
                }
                if (yDeltaTotal < 0)
                    yDeltaTotal = 0;

                if (mIsBeingDragged) {
                    scrollTo(0, yDeltaTotal);
                }

                mLastX = x;
                mLastY = y;
                break;
        }

        return true;
    }

