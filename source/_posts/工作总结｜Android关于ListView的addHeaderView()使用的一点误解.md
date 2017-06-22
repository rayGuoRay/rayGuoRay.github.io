
![](http://upload-images.jianshu.io/upload_images/1489253-62e6f734b47199b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 在日常的开发中在ListView中添加Header是一种很常见的需求,猿就日前关于addHeaderView中的一点总结拿出来分享给大家。

目前对于ListView中addHeaderView()方法的使用大致有以下两种说法:
1. addHeaderView()方法必须在setAdapter()之前进行,否则会抛出异常。
2. addHeaderView()方法可以随时调用。

在经过分析后,猿发现:**以上两种说法都是正确的╮(╯▽╰)╭, 原因就在于Android版本不一致。**由于比对所有源码工作量较大,所以我们就选取三个Android版本进行比较,分别是:Android ICE_CREAM_SANDWICH_MR1(VersionCode  = 15)、Android L(VersionCode = 21)、Android N(VersionCode = 24)。

##### ICE_CREAM_SANDWICH_MR1版本中的部分实现:
- addHeaderView()方法部分关键实现

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

- setAdapter()方法实现

      if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
          mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
      } else {
          mAdapter = adapter;
      }
      
我们可以看到，在这个版本中addHeaderView时，会对ListView绑定的adapter进行判断，如果非空并且类型不为HeaderViewListAdapter时，就会抛出IllegalStateException。而对于HeaderViewListAdapter的来源，则是在setAdapter方法中处理，如果当前有Header，则会将adapter包装为HeaderViewListAdapter，否则的话，就不进行处理，所以**在这个版本中，"addHeaderView()方法必须在setAdapter()之前进行,否则会抛出异常"这种说法是正确的**。

#####Android L版本中的部分实现:
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

#####Android N版本中的部分实现:
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

- setAdapter()方法实现

      if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
         mAdapter = wrapHeaderListAdapterInternal(mHeaderViewInfos, mFooterViewInfos, adapter);
      } else {
         mAdapter = adapter;
      }

而对于Android L 和Android N这两个版本,其基本实现逻辑一致，相比于ICE_CREAM_SANDWICH_MR1版本,对于在setAdapter之后调用addHeaderView这种情况，并不会抛出异常，反而会对mAdapter强制进行一层包装，保证addHeaderView的Adapter是HeaderViewListAdapter，之后通知数据进行刷新。所以**“addHeaderView()可以随时调用”这种说法在这两个版本中其实也是正确的**。

*事实上考虑到目前大多数手机都运行在4.0以上的机型，所以对于addHeaderView()方法的使用本猿更倾向于第二种*

以上~~~b(￣▽￣)d
