---
title: Android N DownloadManager 源码分析
date: 2016-09-02 22:18:56
categories: 源码分析
tags: [Android, DownloadManager]
keywords: Android, DownloadManager
comments: true
description: DownloadManager 是 Android 用系统服务的方式提供的用来优化处理长时间下载任务的工具。本文将基于 Android N 的源码进行分析。
---

DownloadManager 是 Android 用系统服务的方式提供的用来优化处理长时间下载任务的工具。本文将基于 Android N 的源码进行分析。

### DownloadManager的使用方式
    DownloadManager downloadManager = (DownloadManager) context.getSystemService(Context.DOWNLOAD_SERVICE);
    Uri uri = Uri.parse("downloadUrl");
    DownloadManager.Request request = new Request(uri);
    long reference = downloadManager.enqueue(request);

调用enqueue方法之后，只要数据连接可用并且Download Manager可用，下载就会开始。
要在下载完成的时候获得一个系统通知（notification）,注册一个广播接受者来接收ACTION_DOWNLOAD_COMPLETE广播，这个广播会包含一个EXTRA_DOWNLOAD_ID信息在intent中包含了已经完成的这个下载的ID。

*其他更详细API使用方法请参考[Android DownloadManager的使用](http://blog.csdn.net/sir_zeng/article/details/8983430)一文,此处不再详述。*

### DownloadManager的调用处理

DownloadManager的执行入口方法enqueue的源码如下所示：

    ContentValues values = request.toContentValues(mPackageName);
    Uri downloadUri = mResolver.insert(Downloads.Impl.CONTENT_URI, values);
    long id = Long.parseLong(downloadUri.getLastPathSegment());
    return id;

其中，request为请求初始化传入的DownloadManager.Rquest对象，传入请求后
toContentValues()方法会以传入包名将待插入的数据生成ContentValues,方法中会有一个断言检查，代码如下所示：

    ContentValues toContentValues(String packageName) {
        ContentValues values = new ContentValues();
        assert mUri != null;
        //.......
    }

*其实看到这处断言检查有点疑惑，在构造Uri对象的时候已经进行了空判断，为什么此处还要进行一次断言检查呢,不是会有冗余吗?*

在插入ContentValues时，mResolver.insert()实际调用的是系统DownloadProvider中的insert方法,插入返回的downloadUri会在原有Uri基础上调用`ContentUris.withAppendedId(Downloads.Impl.CONTENT_URI, rowID)`添加一个rowId返回一个形如`content://downloads/my_downloads/33`的Uri，经过Uri截取之后，实际操作的reference其实是数据库中的rowId(数据库行号)。

### DownloadProvider的调用处理
在之前版本中，DownloadProvider在插入数据后，会直接以context.startService的方式
来启动DownloadService。进行异步任务下载。而在Android N版本中引入了JobSchedule组件来进行异步下载任务的处理。
在Android L版本中引入的JobScheduler可以控制耗电，具体使用可以参考：[Android JobSchedule工作调度](http://blog.csdn.net/qq_31726827/article/details/50462025),
其中DownlaodProvider中的insert方法中的关键操作如下所示：

    final long token = Binder.clearCallingIdentity();
    try {
      Helpers.scheduleJob(getContext(), rowID);
    } finally {
      Binder.restoreCallingIdentity(token);
    }

其中Helpers.scheduleJob()方法中使用rowId将那条下载信息查询出来，然后调用绑定的DownloadJobService进行下载任务。如果线程调度失败，会返回false。

    public static void scheduleJob(Context context, long downloadId) {
        final boolean scheduled = scheduleJob(context,　DownloadInfo.queryDownloadInfo(context, downloadId));
        if (!scheduled) {
            // If we didn't schedule a future job, kick off a notification
            // update pass immediately
            getDownloadNotifier(context).update();
        }
    }
此时getDownloadNotifier(context).update()会将遍历出所有未删除的

### DownloadJobService调度执行
DownloadService中调度的线程开始下载，在onStartJob中用rowId查出来后，直接开线程开始下载，具体代码如下所示：

    public boolean onStartJob(JobParameters params) {
      final int id = params.getJobId();
      // Spin up thread to handle this download
      final DownloadInfo info = DownloadInfo.queryDownloadInfo(this, id);
      if (info == null) {
          Log.w(TAG, "Odd, no details found for download " + id);
          return false;
        }
        final DownloadThread thread;
        synchronized (mActiveThreads) {
          thread = new DownloadThread(this, params, info);
          mActiveThreads.put(id, thread);
        }
        thread.start();
        return true;
    }

### DownloadJobService中的暂停、取消与完成
DownloadJobService中在线程开启后，会刷新展示相应的通知栏，通过通知栏UI中的相应控制，可以实现对于下载任务的控制。
- 在开始下载后，当点击取消后，会发送广播到DownlaodReceiver,当接受到这个广播后，会调用DownloadManager.remove(downloadIds)，而DownloadManager.remove()方法则会调用DownloadProvider.delete去删除记录任务。同时会依据rowId移除该线程调度。

- 任务完成时，会发送一个广播，通知下载完成，但是这里比较意外的是，下载完成的广播发送是放在DownloadInfo中调用DownloadInfo.sendIntentIfRequested()发送的， 而不是在DownloadThread中。

- 暂停，比较奇怪的是，DownloadManager的异步下载线程提供了断点下载的功能，写入文件也会检查任务的下载状态是不是暂停，但是，却并未提供暂停下载任务的API方法，同时它的下载状态查询的方法也是私有类型的。如果需要暂停任务就需要自定义自己的下载任务了。

### DownloadThread中的断点下载的实现方法
其实在DownloadThread中，主要的下载方法就是就是线程中的excuteDownload()方法。部分关键代码如下：

    private void executeDownload() throws StopRequestException {
      final boolean resuming = mInfoDelta.mCurrentBytes != 0;
      ...
      int redirectionCount = 0;
      while (redirectionCount++ < Constants.MAX_REDIRECTS) {
          ......
          conn = (HttpURLConnection) mNetwork.openConnection(url);
          addRequestHeaders(conn, resuming);
          final int responseCode = conn.getResponseCode();
          switch (responseCode) {
              case HTTP_OK:
                  if (resuming) {
                      throw new StopRequestException(
                              STATUS_CANNOT_RESUME, "Expected partial, but received OK");
                  }
                  parseOkHeaders(conn);
                  transferData(conn);
                  return;
              case HTTP_PARTIAL:
                  if (!resuming) {
                      throw new StopRequestException(
                              STATUS_CANNOT_RESUME, "Expected OK, but received partial");
                  }
                  transferData(conn);
                  return;
              ......
          }
          ......
      }
    }

在addRequestHeaders()方法中，如果从数据库中查出的数据已读取写入文件的字节数不为0，则会在请求头前添加一个range`conn.addRequestProperty("Range", "bytes=" + mInfoDelta.mCurrentBytes + "-");`，当添加上此请求头后，当求求成功后，服务器会返回HTTP_PARTIAL,将接收到的数据通过transferData()方法写入到文件中。在写入文件中时，DownloadThread引入了android.drm.DrmManagerClient与android.drm.DrmOutputStream，这两个包位于framework/base/core/drm包下，部分引用代码如下所示：

    if (DownloadDrmHelper.isDrmConvertNeeded(mInfoDelta.mMimeType)) {
      drmClient = new DrmManagerClient(mContext);
      out = new DrmOutputStream(drmClient, outPfd, mInfoDelta.mMimeType);
    } else {
      out = new ParcelFileDescriptor.AutoCloseOutputStream(outPfd);
    }
*对于这两个类的引入，目前还不是特别熟悉，后续研究后会进一步进行分析*

最后丧心病狂的自己画个图，简单总结下DownloadManager的工作流程:整体外源应用层通过FrameWork层DownloadManager API调用到DownloadProvider,通过操作数据库，最后通过DownloadService中的线程调度完成工作。整体上都是由DownloadProvider进行过渡调用。而数据库与Service都通过DownloadProvider进行隔离。


![简单结构图](http://upload-images.jianshu.io/upload_images/1489253-eb335cfacbcd097b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> DownloadManager中的分析目前就先告一段落，文中如有分析错误或描述不清楚之处，请大家留言指出～:）
