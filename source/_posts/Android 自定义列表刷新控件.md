---
title: Android 列表刷新控件 EasyRefreshView
date: 2017-05-121 00:18:56
categories: 工具分享
tags: [Android, RecyclerView, PullToRefresh, EasyRefreshView]
keywords: Android, RecyclerView, PullToRefresh, EasyRefreshView
comments: true
description: Android 中上拉加载和下拉刷新都是很常用的控件，所以在 Android 后续版本中提供了 PullToRefresh 这个控件，以方便开发者很便捷的集成下拉刷新功能。而对于上拉加载功能，仍然需要开发者自己监听 ListView 或者 RecyclerView 滑动状态来实现自己的上拉加载功能。因此在最近猿最近基于 PullToRefresh 与 RecyclerView 自己试做了一个控件：EasyRefreshView，来方便开发者集成相关功能。
---

Android 中上拉加载和下拉刷新都是很常用的控件，所以在 Android 后续版本中提供了 PullToRefresh 这个控件，以方便开发者很便捷的集成下拉刷新功能。而对于上拉加载功能，仍然需要开发者自己监听 ListView 或者 RecyclerView 滑动状态来实现自己的上拉加载功能。因此在最近猿最近基于 PullToRefresh 与 RecyclerView 自己试做了一个控件：EasyRefreshView，来方便开发者集成相关功能。

### 控件截图

![](http://upload-images.jianshu.io/upload_images/1489253-0220580c36c04f7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 使用说明
1、AndroidStudio 集成方法

        compile 'com.ray.easyrefreshview:easy-refresh-view:0.5.9'

2、Layout 布局文件声明

        <com.ray.easyrefreshview.EasyRefreshView
            android:id="@+id/user_elv"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            easyrefreshview:layoutType="linear"
            easyrefreshview:normalLayout="@layout/item_normal"
            easyrefreshview:loadingLayout="@layout/foot_loading"
            easyrefreshview:nomoreLayout="@layout/foot_loading_nomore_data"
            easyrefreshview:errorLayout="@layout/foot_loading_error"/>

控件支持用户自定义相关 Item Layout ，分别是正常显示状态、等待显示状态、无更多数据状态、加载错误状态，集成者可以在 bindAdapter 中的 onCreate 回调方法中去创建对应 layout 的 Holder，并在 onBind 回调方法对相应的 View 进行操作。

3、控件绑定 Adapter 并自定义相关 ViewHolder

        easyRefreshView.bindAdapter(new EasyRefreshHolderCallBack() {
            @Override
            public RecyclerView.ViewHolder onCreateNormal(View view) {
                //return new NormalHolder(view);
            }

            @Override
            public void onBindNormal(RecyclerView.ViewHolder holder, int position) {
                super.onBindNormal(holder, position);
                //todo
            }
        });

4、 控件滑动到顶部和底部的两种回调

当列表滑动到顶部和底部时，分别会有 onTopLoadStarted 与 onBottomLoadStarted 回调触发，继承者可以在这两个回调中进行下拉加载与上拉刷新进行数据更新。

        public void onTopLoadStarted() {
            list.clear();
            startRxLoad(0);
        }

        public void onBottomLoadStarted(int position) {
            if (position >= mTotalCount) {
                erv.setFootViewState(EasyRefreshAdapter.FOOT_STATE_LOAD_NOMORE);
                return;
            }
            erv.setFootViewState(EasyRefreshAdapter.FOOT_STATE_LOADING);
            startRxLoad(position + 1);
        }
目前控件已开源至GitHub([相关链接](https://github.com/rayGuoRay/RssMovie)) , 相关 Demo 使用数据来源豆瓣电影开放 API ,使用 Retrofit2 进行网络请求，并利用RxJava 对进行了部分逻辑处理，欢迎 Fork 和 Star ，如果有改进意见也可以提给猿，猿会在后续版本中加以改进。

O(∩_∩)O谢谢
