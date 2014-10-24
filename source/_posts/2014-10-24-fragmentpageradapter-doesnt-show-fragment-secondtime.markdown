---
layout: post
title: "使用FragmentPagerAdapter的fragment第二次打开不能显示内容的问题分析"
date: 2014-10-24 11:03:20 +0800
comments: true
categories: android FragmentPagerAdapter fragment lj
---

在使用viewpager的时候有两个adapter：FragmentPagerAdapter和FragmentStatePagerAdapter，区别在于他们的instantiateItem方法是不一样的：

FragmentPagerAdapter

```java

    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {

            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }
```




FragmentStatePagerAdapter

```java

    public Object instantiateItem(ViewGroup container, int position) {
        // If we already have this item instantiated, there is nothing
        // to do.  This can happen when we are restoring the entire pager
        // from its saved state, where the fragment manager has already
        // taken care of restoring the fragments we previously had instantiated.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        mFragments.set(position, fragment);
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }
```

FragmentPagerAdapter会将adapter的每个fragment加入fragmentManager管理，如果初始化adapter传的是getSupportFragmentManager(),则每个fragment都会放到activity的fragmentManager里面去。这样就会造成当viewpager所在的fragment因为popFromBackStack被移除后，再次add一次viewpager所在的fragment，viewpager里面不会显示出来fragment内容。原因是这样的：

第一次FragmentPagerAdapter的instantiateItem方法会把每个fragment加入activity的fragmentManager当中，popFromBackStack这个viewpager所在的fragment时，并不会将viewpager里面的每个fragment调用detach。第二次就会发现fragment已经存在，则直接调用mCurTransaction.attach方法，而attachFragment方法缺发现fragment未被detach则直接跳过了：


```java

    public void attachFragment(Fragment fragment, int transition, int transitionStyle) {
        if (DEBUG) Log.v(TAG, "attach: " + fragment);
        if (fragment.mDetached) {
            fragment.mDetached = false;
            if (!fragment.mAdded) {
                if (mAdded == null) {
                    mAdded = new ArrayList<Fragment>();
                }
                if (mAdded.contains(fragment)) {
                    throw new IllegalStateException("Fragment already added: " + fragment);
                }
                if (DEBUG) Log.v(TAG, "add from attach: " + fragment);
                mAdded.add(fragment);
                fragment.mAdded = true;
                if (fragment.mHasMenu && fragment.mMenuVisible) {
                    mNeedMenuInvalidate = true;
                }
                moveToState(fragment, mCurState, transition, transitionStyle, false);
            }
        }
    }
```


如果使用FragmentStatePagerAdapter则不会出现这个问题，它自己内部使用mFragments数组维持fragment缓存，因为我们重新打开viewpager所在的fragment时，重新初始化了一个新的viewpager adapter，所以其实始终没用到这个缓存，每次都会重新使用getItem来new出来新fragment，然后调用mCurTransaction.add加入。

##要避免这个问题有几种方法：

1.使用FragmentStatePagerAdapter
2.初始化viewpager的adapter的时候不要使用getSupportFragmentManager()，而使用getChildFragmentManager()。这样就会找不到缓存的fragment。
3.或者使用FragmentPagerAdapter，在viewpager所在的fragment被detach的时候显示detach viewpager里面的每一个fragment

FragmentPagerAdapter和FragmentStatePagerAdapter区别在于：前者会保存fragment对象，后者仅会保存fragment的状态能进行恢复，占用内存少一些。

官方也有篇讨论帖：看[这里](https://code.google.com/p/android/issues/detail?id=59579&q=FragmentPagerAdapter%20instantiateItem&colspec=ID%20Type%20Status%20Owner%20Summary%20Stars)


