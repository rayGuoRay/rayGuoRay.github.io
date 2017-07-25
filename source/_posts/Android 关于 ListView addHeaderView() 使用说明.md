---
title: Android 关于 ListView addHeaderView() 使用说明
date: 2017-03-04 12:37:56
categories: 工作总结
tags: [Android, ListView, HeaderView]
keywords: Android, ListView, HeaderView
comments: true
description: 目前 Android 中对于 ListView 中 addHeaderView() 方法的使用大致有以下两种说法，一种需要在 setAdapter() 之前调用，否则会抛出异常。另外一种则随时可以调用，不会抛出异常。猿在使用中有些疑惑，经过分析，发现两种说法都是正确的。
---

目前 Android 中对于 ListView 中 addHeaderView() 方法的使用大致有以下两种说法:
1. addHeaderView() 方法必须在 setAdapter() 之前进行,否则会抛出异常。
2. addHeaderView() 方法可以随时调用。

在经过分析后,猿发现:**以上两种说法都是正确的╮(╯▽╰)╭, 原因就在于Android版本不一致。**由于比对所有源码工作量较大,所以我们就选取三个Android版本进行比较,分别是:Android ICE_CREAM_SANDWICH_MR1(VersionCode  = 15)、Android L(VersionCode = 21)、Android N(VersionCode = 24)。**

### ICE_CREAM_SANDWICH_MR1 版本中的部分实现:
- addHeaderView() 方法部分关键实现

      public void addHeaderView(View v, Object data, boolean isSelectable) {

        if (mAdapter != null && ! (mAdapter instanceof HeaderViewListAdapter)) {
            throw new IllegalStateException(
                    "Cannot add header view to list -- setAdapter has already been called.");
        }

        FixedViewInfo info = new FixedViewInfo();
        info.view = v;
        info.data = data;
        info.isSelectable = isSelectable;
        mHeaderViewInfos.add(info);

        // in the case of re-adding a header view, or adding one later on,
        // we need to notify the observer
        if (mAdapter != null && mDataSetObserver != null) {
            mDataSetObserver.onChanged();
        }
      }

- setAdapter() 方法实现

      if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
          mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
      } else {
          mAdapter = adapter;
      }

我们可以看到，在这个版本中addHeaderView时，会对ListView绑定的adapter进行判断，如果非空并且类型不为HeaderViewListAdapter时，就会抛出IllegalStateException。而对于HeaderViewListAdapter的来源，则是在setAdapter方法中处理，如果当前有Header，则会将adapter包装为HeaderViewListAdapter，否则的话，就不进行处理，所以**在这个版本中，"addHeaderView()方法必须在setAdapter()之前进行,否则会抛出异常"这种说法是正确的**。

###Android L 版本中的部分实现:
- addHeaderView()方法部分关键实现

      public void addHeaderView(View v, Object data, boolean isSelectable) {
        final FixedViewInfo info = new FixedViewInfo();
        info.view = v;
        info.data = data;
        info.isSelectable = isSelectable;
        mHeaderViewInfos.add(info);
        mAreAllItemsSelectable &= isSelectable;

        // Wrap the adapter if it wasn't already wrapped.
        if (mAdapter != null) {
            if (!(mAdapter instanceof HeaderViewListAdapter)) {
                mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, mAdapter);
            }

            // In the case of re-adding a header view, or adding one later on,
            // we need to notify the observer.
            if (mDataSetObserver != null) {
                mDataSetObserver.onChanged();
            }
        }
      }

- setAdapter()方法实现

      if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
         mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
      } else {
         mAdapter = adapter;
      }

###Android N 版本中的部分实现:
- addHeaderView()方法部分关键实现

      public void addHeaderView(View v, Object data, boolean isSelectable) {
        final FixedViewInfo info = new FixedViewInfo();
        info.view = v;
        info.data = data;
        info.isSelectable = isSelectable;
        mHeaderViewInfos.add(info);
        mAreAllItemsSelectable &= isSelectable;

        // Wrap the adapter if it wasn't already wrapped.
        if (mAdapter != null) {
            if (!(mAdapter instanceof HeaderViewListAdapter)) {
                wrapHeaderListAdapterInternal();
            }

            // In the case of re-adding a header view, or adding one later on,
            // we need to notify the observer.
            if (mDataSetObserver != null) {
                mDataSetObserver.onChanged();
            }
        }
      }

- setAdapter() 方法实现

      if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
         mAdapter = wrapHeaderListAdapterInternal(mHeaderViewInfos, mFooterViewInfos, adapter);
      } else {
         mAdapter = adapter;
      }

而对于 Android L 和 Android N 这两个版本,其基本实现逻辑一致，相比于 ICE_CREAM_SANDWICH_MR1 版本,对于在 setAdapter 之后调用 addHeaderView 这种情况，并不会抛出异常，反而会对 mAdapter 强制进行一层包装，保证 addHeaderView 的 Adapter 是 HeaderViewListAdapter ，之后通知数据进行刷新。所以**“ addHeaderView() 可以随时调用”这种说法在这两个版本中其实也是正确的**。

*事实上考虑到目前大多数手机都运行在4.0以上的机型，所以对于 addHeaderView() 方法的使用本猿更倾向于第二种*

以上~~~b(￣▽￣)d
