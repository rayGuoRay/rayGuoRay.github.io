![](http://upload-images.jianshu.io/upload_images/1489253-0d39e4bbdf7ba28d.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 在之前的文章中本猿曾介绍过Android N DownloadManager中已经采用了JobSchedule方式进行下载任务的调度,今天就开发过程中碰到的关于JobSchedule的一点小坑,总结出来供大家参考分析。

想了解更多关于Android N中DownloadManager的源码分析请点击下文:
- [源码学习｜Android N DownloadManager源码分析](http://www.jianshu.com/p/c9dc04af2f54)

##### 问题背景
在项目中,项目采用基于源码修改的DownloadManager进行下载任务,而出于控制下载流量的考虑,会额外对网络访问权限进行一定的处理。目前主要修改点有两个:
1. Helpers类中的scheduleJob方法,主要作用是通过schedule来唤起DownloadJobService,最终执行下载操作。
2. DownloadInfo类中的getRequiredNetworkType()方法,主要作用来返回下载任务需要的网络类型。

##### 问题描述
当前存在的问题是对于Android 7.0的手机共享出来的热点,我们的下载任务一直处于等待下载状态,无法执行。

##### 问题结果
在前文已经介绍过DownloadManager启动DownloadJobService是通过Helpers.scheduleJob()方法来调度任务,拉起DownloadJobService,最终完成下载功能的,
对于目前存在的问题,我们发现问题原因是由于scheduler.scheduleAsPackage()已经完成了调度工作,并返回了调度值为1。但为什么没有调用DownloadJobService呢,我们最后还是注意到我们设置的网络类型:

        // We always require a network, but the type of network might be further
        // restricted based on download request or user override
        builder.setRequiredNetworkType(info.getRequiredNetworkType(info.mTotalBytes));

我们要注意到,此处setRequiredNetworkType设置了调度任务执行需要的网络类型,而当前DonwloadInfo连接手机热点获取的网络类型为WIFI,由于需求要求控制访问网络请求的原因,所以我们对于所有网络类型为WIFI的网络,返回的类型为JobInfo.NETWORK_TYPE_UNMETERED,即只允许访问非计费的网络。

但是对于Android6.0与7.0版本的手机,开出的热点识别出的NetworkInfo中mIsMetered对象为true, 而对于普通wifi中对于mIsMetered对象为false,这个标记值是用来标记当前网络是否为计费网络的标记位。对于手机网络的热点,此处他会识别为计费的网络,而对于普通通过路由分出的热点,会识别为非计费的网络,因此针对这个问题,我们将需要的网络类型设置为JobInfo.NETWORK_TYPE_ANY即可正常完成调用。

##### 问题后记
我们从这里可以看出Android系统设计的细腻之处,对于访问网络更强大的支持可以更灵活的让我们对手机网络的访问进行控制。

但有个最重要的事是:对于是否为手机热点还不能单纯的以这个字段来判断,因为从Iphone上开出的热点,当手机读取的时候,这个mIsMetered对象又变成了true。

**所以当有产品经理问你能不能识别当前连接的是wifi还是手机热点时,千万别急着说可以识别哦~~O(∩_∩)O哈哈~**

*大家对于Android 网络请求有什么建议和见解欢迎留言指导~~~*
