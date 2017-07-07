---
title: Volley 网络库源码初探
date: 2016-09-29 17:41:56
categories: 源码分析
tags: [源码分析, Android, Volley]
keywords: 源码分析, Android, Volley
comments: true
---

Volley是Google I/O 2013上发布的一个网络通信库，本文将基于Ａndroid N Frameworks层中的Volley源码对其基础实现进行分析。

### Volley 框架介绍:
在计算机网络发展中，诞生了两种经典的计算机网络参考模型： OSI 参考模型与 TCP/IP 参考模型，其中的分层如下所示：

|OSI 七层模型| TCP/IP 四层模型|传输的数据|
|:-:|:-:|:-:|
| 应用层|应用层|数据|
|表示层|应用层|数据|
|会话层|应用层|数据|
|传输层|传输层|段|
|网络层|网际互联层|包|
|数据链路层|网络接入层|帧|
|物理层|网络接入层|比特流|

而目前　Android　开发中常提及的各种网络框架面向的网络层也是不同的，针对 TCP/IP 四层模型中，现在使用较多的 OKHttp ，HttpClient ，URLConnection 都是面向传输层的网络框架，而 Retrofit ， AsyncHttp ，Volley 则是依托于这些面向传输层的网络框架。其中 Retrofit 中默认采用 OKHttp 作为其网络传输层 ，AsyncHttp 则默认采用 HttpClient ，而 Volley 中对于 API Level 大于8的采用 URLConnection 作为网络传输层,而小于等于8的采用 HttpClient 作为网络传输层,同时它支持定义 OKHttp 作为其网络传输层。**对于一些说 Volley 是基于 OKHttp 进行封装的网络库,个人认为这种说法其实并不准确。**

### Volley 简单使用

Volley 的使用大致分为三步:请求初始化、构建请求参数、设置请求回调、执行请求。如下所示是简单的 Volley 请求写法。

    RequestQueue mQueue = Volley.newRequestQueue(context);  
    StringRequest stringRequest = new StringRequest(url,  
                    new Response.Listener<String>() {  
                        @Override  
                        public void onResponse(String response) {  
                            Log.d("TAG", response);  
                        }  
                    }, new Response.ErrorListener() {  
                        @Override  
                        public void onErrorResponse(VolleyError error) {  
                            Log.e("TAG", error.getMessage(), error);  
                        }  
                    });  
    mQueue.add(stringRequest);  

### Volley 源码分析

Android 7.0 源码中关于 Volley 的库位于 frameworks/volley 包下。现我们将按 Volley 的使用步骤进行源码的分析:

#### Volly RequestQueue 的初始化

Volley 调用的 newRequestQueue 有两类构造方法:

- newRequestQueue(Context context)
- newRequestQueue(Context context, HttpStack stack)

如果不选择传入 HttpStack ,则 Volley 将以传入 null 的形式调用后一方法。其具体实现如下所示:

    File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
    String userAgent = "volley/0";
    try {
        String packageName = context.getPackageName();
        PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
        userAgent = packageName + "/" + info.versionCode;
    } catch (NameNotFoundException e) {
    }
    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            stack = new HurlStack();
        } else {
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
        }
    }
    Network network = new BasicNetwork(stack);
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
    queue.start();

我们可以看到,对于 stack 的处理:如果默认传入 HttpStack 的实例 stack 为空,则会根据 SDK API 版本选择不同的实现类.其中 HurlStack 与 HttpClientStack 都是 HttpStack 的实现类,从名称上以可以看的出来, HurlStack 是封装的 HttpURLConnection，而　HttpClientStack 是封装的 HttpClient。其具体的封装与请求使用我会在其他文章中进行详细分析。此处不再详述。

*BasicNetWork 是 Network 接口的实现类,其中接口的实现方法 performRequest(Request<?> request) 中会调用请求 Request 去真正执行网络请求。此处的 start() 方法只是调用 RequestQueue的start() 方法启动 CacheDispatcher 与 NetRequestDispatcher 两个请求转发线程,具体请参考后文的 RequestQueue 的 start 方法分析。*

#### Volley RequestQueue 的构造函数
在 RequestQueue 中 start() 方法启动前,会调用 RequestQueue 构造方法进行初始化,将 NetWork 实现类和默认的缓存传入 RequestQueue 中,完成 RequestQueue 的初始化。其中最终调用的初始化方法如下所示:

    public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;
        mNetwork = network;
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }

在构造方法中, RequestQueue 中对传入各部分的参数注释如下:

- cache A Cache to use for persisting responses to disk (Cache 是一个接口定义,默认使用DiskCache来持久化响应结果)
- network A Network interface for performing HTTP requests (Network 是一个接口定义,默认使用BasicNetwork来执行HTTP请求)
- threadPoolSize Number of network dispatcher threads to create (使用NetworkDispatcher数组定义执行器容器,默认容器最大容量为4)
- delivery A ResponseDelivery interface for posting responses and errors (请求结果转发接口,默认使用ExecutorDelivery转发结果)

在构造函数之前的注释说明的更清楚一点: Creates the worker pool. Processing will not begin until {@link #start()} is called. 大意为创建工作池,但是在调用 start 方法之前,整个请求并不会真正开始工作。那么接下来我们便来分析 RequestQueue 的启动方法 start() 。

#### Volley RequestQueue 的启动

RequestQueue 中的 start() 方法中,会真正的开始执行一个请求队列,而在这之中, RequestQueue 又会引入二个新的类型,分别为: CacheDispatcher 与 NetworkDispatcher ,具体代码如下所示:

    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

在启动任务之前会调用 stop() 方法确保当前没有正在运行任务这个比较好理解,而接下来创建的 CacheDispatcher 从构造方法传参看,其中又多了一个未曾在前面出现过的 mCacheQueue , NetWorkDispatcher 也多了一个未曾在前面出现过的 mNetworkQueue。这两个集合类是在类变量声明时便进行初始化了:

    private final PriorityBlockingQueue<Request<?>> mCacheQueue = new PriorityBlockingQueue<Request<?>>();
    private final PriorityBlockingQueue<Request<?>> mNetworkQueue = new PriorityBlockingQueue<Request<?>>();

CacheDispatcher 继承自 Thread,从名称就可以看出,主要作用是管理缓存派发请求的。而 NetworkDispatcher 同样也继承自 Thread,是处理缓存未派发的请求的。当调用其 start() 方法后,线程便会启动。我会在其他文章中对它的缓存派发和请求管理进行详细分析。此处不再详述

#### Volley 构建 Request 请求与注册监听

在 Volley 对 RequestQueue 初始化完毕后,我们需要构建请求,来完成我们的网络请求.在简单使用的代码中,构建请求使用的是 StringRequest,它是抽象类 Request 的子类。Request 的子类请求还有 ImageRequest、JsonRequest。而 JsonRequest 是 JsonArrayRequest 与 JsonObjectRequest 的父类。

对于 Request 类中,我们注意到,对于 Request 这个顶级父类,其监听只注册了一个 ErrorListener,而结果监听则交给其实现类去注册,如 StringRequest 的结果监听为 Listener<String> 、ImageRequest 的结果监听为 Listener<Bitmap>、JsonArrayRequest注册的结果监听为 Listener<JSONArray>、JsonObjectRequest 注册的结果监听为 Listener<JSONObject>。而 JsonRequest 的结果监听却支持范型传入 Listener<T>。

**从这点来说,个人有些疑惑,既然支持范型定义,为何不直接在顶级父类定义封装,然后在子类中传入参数进行初始化使用呢? 从软件分层的角度来看,好像这样也更合理。希望对这个问题有想法的同学能提供些思路给我。**

而对于不同的 Request 子类实现类,笔者会在以后对几种实现类进行详细分析。大致概括说来, Volley 会对请求结果进行解析封装,以对应的类型包装回调回来,其是由抽象类中的 deliverResponse() 返回,其代码如下所示:

    @Override
    protected void deliverResponse(String response) {
        mListener.onResponse(response);
    }

而在 **RequestQueue 的初始化** 中我们说过, BasicNetWork 中的 performRequest() 方法是会去执行 Request 的网络请求,那它是要如何请求呢,这就需要我们将构建的请求与 ReqeustQueue 及其相关组件关联起来了,这就是我们下一步要介绍的 RequestQueue 的 add() 方法。

#### RequestQueue 添加 Request 请求

在使用对应的 Request 构建好网络请求后, RequestQueue 会调用其 add() 方法将构建的请求加入到请求队列中,其具体执行方法代码如下所示:

    // If the request is uncacheable, skip the cache queue and go straight to the network.
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
    // Insert request into stage if there's already a request with the same cache key in flight.
    synchronized (mWaitingRequests) {
        String cacheKey = request.getCacheKey();
        if (mWaitingRequests.containsKey(cacheKey)) {
            // There is already a request in flight. Queue up.
            Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
            if (stagedRequests == null) {
                stagedRequests = new LinkedList<Request<?>>();
            }
            stagedRequests.add(request);
            mWaitingRequests.put(cacheKey, stagedRequests);
            if (VolleyLog.DEBUG) {
                VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
            }
        } else {
            // Insert 'null' queue for this cacheKey, indicating there is now a request in
            // flight.
            mWaitingRequests.put(cacheKey, null);
            mCacheQueue.add(request);
        }
        return request;
    }

从代码中可以看到,如果 Request 不允许缓存的话, Request 直接加入到 NetWorkQueue 中就直接 return 了,而默认 request 是允许使用缓存的,当允许使用缓存时,如果 WaitingRequestsQueue 中没有相同的请求任务,则会将任务 key 加入到 waitingRquests 中,但是传入队列为空,然后将任务加入 CacheQueue 中。如果已经有同样的请求在运行,则会将新任务构建队列,加入到 WaitingRequest 中,但并不不执行。

将 Request 加入到 CacheQueue 中后,我们留意到 CacheQueue 后续没有操作了,这个原因在于,CacheQueue 与前文中描述的 CacheDispatcher 耦合, CacheDispatcher 作为一个线程死循环,在 RequestQueue 中已经启动,它会不停的遍历 CacheQueue 中的 Request 请求。其部分主要代码如下所示:

    // Attempt to retrieve this item from cache.
    Cache.Entry entry = mCache.get(request.getCacheKey());
    if (entry == null) {
        request.addMarker("cache-miss");
        // Cache miss; send off to the network dispatcher.
        mNetworkQueue.put(request);
        continue;
    }

    // If it is completely expired, just send it to the network.
    if (entry.isExpired()) {
        request.addMarker("cache-hit-expired");
        request.setCacheEntry(entry);
        mNetworkQueue.put(request);
        continue;
    }

    // We have a cache hit; parse its data for delivery back to the request.
    request.addMarker("cache-hit");
    Response<?> response = request.parseNetworkResponse(
            new NetworkResponse(entry.data, entry.responseHeaders));
    request.addMarker("cache-hit-parsed");

    if (!entry.refreshNeeded()) {
        // Completely unexpired cache hit. Just deliver the response.
        mDelivery.postResponse(request, response);
    }

如果缓存未命中或者缓存过期,则会调用 NetworkQueue 进行请求。如果缓存命中,则会直接调用 ResponseDelivery (默认为其实现类 ExecutorDelivery )进行结果分发。对于 NetworkQueue.post(Request) 的执行结果,最后会由 NetworkDispatcher 去执行,最后也会将执行结果通过 ResponseDelivery 调用 postResponse(reqeust, response) 将结果返回。

**特别的对于 reponse 的结果 CacheDispatcher 和 NetworkDispathcer 都会调用 Request.ParseNetworkResponse() 方法来解析 Response 结果。这样处理的结果就是,在回调 listenter 中, Response 回调的就是已经处理过的类型,而不用再去做一次数据解析,相关示例会在对于图片请求源码分析中进行说明。**

#### Volley Request 的请求回调

在 ExecutorDelivery 调用 postResponse 方法后, ExecutorDelivery 将以 handler.post(runnable) 的形式来执行调用的 request 与 response ,其关键构建方法如下:

    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

其中,ExecutorDelivery 调用的 postResponse 方法中所执行的 Runnable 对象为内部类 ResponseDeliveryRunnable ,其执行工作主要是将结果回调到 Request 中,其关键代码如下所示:

    // Deliver a normal response or error, depending.
    if (mResponse.isSuccess()) {
        mRequest.deliverResponse(mResponse.result);
    } else {
        mRequest.deliverError(mResponse.error);
    }

在回调到 Request 中后,最后会通过祖册的 listener 将 result 回调回来,以 StringRequest 如下所示:

    @Override
    protected void deliverResponse(String response) {
        mListener.onResponse(response);
    }

这样一个完整的 Volley 网络请求也就完成了。*最后附上一张简单的流程图,随着分析完善,图中缺失的部分也会慢慢补齐*

![Volley 简单请求流程图](http://upload-images.jianshu.io/upload_images/1489253-1b1a6b2aee03f6f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*对于本文中的描述有什么疑问或者建议都可以留言提出,欢迎大家批评指导。O(∩_∩)O哈哈哈~*

> 对于Volley的初步源码分析我们也暂告一段落,在这篇文章中所遗留的Volley的异步图片加载支持、缓存策略、Request子类分类请求区别以及Volley采用OkHttp作为网络传输层等都会在后面文章继续分析。

- Volley 源码解析相关链接：
- [Volley 图片加载源码分析](http://www.jianshu.com/p/2aa373c1c624)
- [Volley 网络请求缓存策略源码分析](http://www.jianshu.com/p/dd81b0e692c7)
