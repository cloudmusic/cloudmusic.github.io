---
layout: post
title: "android子view的pressed状态受父view的clickable影响分析"
date: 2014-10-10 14:36:17 +0800
comments: true
categories: lj android
---
android从sdk16开始，修改了onTouchEvent的ACTION_DOWN代码([Android: Child elements sharing pressed state with their parent even when duplicateParentState specified][1])，如下：

sdk16以前：
```java
	case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    mPrivateFlags |= PRESSED; //comment by bran
                    refreshDrawableState();
                    checkForLongClick(0);
                }
                break;

sdk16及以后：
```java
	case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true);
                    checkForLongClick(0);
                }
                break;
```

后者会调用setPressed方法，代码如下：
```java

    public void setPressed(boolean pressed) {
        final boolean needsRefresh = pressed != ((mPrivateFlags & PFLAG_PRESSED) == PFLAG_PRESSED);

        if (pressed) {
            mPrivateFlags |= PFLAG_PRESSED;
        } else {
            mPrivateFlags &= ~PFLAG_PRESSED;
        }

        if (needsRefresh) {
            refreshDrawableState();
        }
        dispatchSetPressed(pressed);
    }
    
    //viewgroup的实现
    protected void dispatchSetPressed(boolean pressed) {
        final View[] children = mChildren;
        final int count = mChildrenCount;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
            // Children that are clickable on their own should not
            // show a pressed state when their parent view does.
            // Clearing a pressed state always propagates.
            if (!pressed || (!child.isClickable() && !child.isLongClickable())) {
                child.setPressed(pressed);
            }
        }
    }
```

可以看出sdk16以后如果子view是!isClickable，父view是isClickable（父view如果是!isClickable都不会处理ACTION_DOWN事件），则即使在子view区域外的点击也会触发子view的pressed行为。这样改的原因不得而知，但个人觉得这不是正常期望的，UI表现上来看，父view点击任何区域子view都会触发pressed。

这种情景是存在的，比如子view是不可点，但又要设置父view可点以防点击父view会触发父父view事件。如果要处理这种情况，父view可以覆盖dispatchSetPressed方法进行重写，或者直接父view设置setEnable(false)。



[1]:http://stackoverflow.com/questions/14179431/android-child-view-sharing-pressed-state-from-its-parent-view-in-jelly-bean
