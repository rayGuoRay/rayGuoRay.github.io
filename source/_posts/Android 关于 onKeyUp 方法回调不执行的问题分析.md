---
title: Android 关于 onKeyUp 方法回调不执行的问题分析
date: 2017-03-10 10:18:56
categories: 工作总结
tags: [Android, ListView, KeyEvent]
keywords: Android, ListView, KeyEvent
comments: true
---

我们大家都知道 Android 中事件有其传递机制，但是对于焦点(Focus)和按键事件，触摸事件之间的联系暂时还没有一个整体的认识。前段时间在修改一个关于按键事件点击的 Bug,让猿对 Android 的事件传递有了更深的了解,现在分享出来给大家。<!--more-->

### 需求描述
当页面中ListView处于选择态时,当第一次点击Back键需要将ListView的选择态清除,点击第二次Back键时,页面才关闭退出。

### 需求实现
在页面Activity中重写了onBackPressed()方法,ListView中重写了View的onKeyUp方法,当ListView中如果是点击态时,在onKeyUp中拦截事件,并清除ListView的选择态,并return True拦截按键事件返回。

### 问题描述
在调试过程中发现:当页面中ListView处于选择态时,点击Back键时,页面直接关闭。程序并不执行ListView中的onKeyUp方法。

### 出现原因
在页面初始化后,在选择ListView为选择态结束后,调用了`ListView.clearFocus()`方法,导致了`ListView的onKeyUp`方法不回调。

### 原因分析
我们大概都知道,Android事件传递从上到下传递,而对于KeyEvent来说,ViewGroup中分发事件方法代码中有这样一段:

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onKeyEvent(event, 1);
        }

        if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
                == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
            if (super.dispatchKeyEvent(event)) {
                return true;
            }
        } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
                == PFLAG_HAS_BOUNDS) {
            if (mFocused.dispatchKeyEvent(event)) {
                return true;
            }
        }

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 1);
        }
        return false;
    }

我们可以看到,在ViewGroup中,会对**mFocused**这个View进行判断,如果它不为空,有焦点,才会由他去分发事件,最终才有可能执行它的回调方法。所以如果我们手动调用clearFocus后,使当前FocusView为空,自然就无法最终执行其回调方法了。

*事实上,当Activity中dispatcherKeyEvent时,是通过Activity中的PhoneWindow绑定的DecoderView进行分发的,DecoderView最终通过ViewGroup的dispatcherKeyEvent将事件分发出去。详细的事件分发流程会在后续文章中进行详细分析。请大家期待O(∩_∩)O*
