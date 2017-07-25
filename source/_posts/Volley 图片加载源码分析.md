---
title: Volley 图片加载源码分析
date: 2016-10-25 19:15:56
categories: 源码分析
tags: [Android, Volley]
keywords: Android, Volley
comments: true
description: Volley 基于基础的网络请求框架封装了自己的图片请求框架，我们在《 Volley 网络库源码初探》这篇文章中已经对 Volley 的网络库工作流程做了进行了简单分析: Volley 中的图片加载方式总结起来有三种，分别为 ImageRequest、ImageLoader、NetworkImageView，这三种方式名称和使用方式各不相同，下面我们将通过每种加载方式的使用来对 Volley的图片的加载框架进行分析。
---

Volley基于基础的网络请求框架封装了自己的图片请求框架，我们在上一篇中已经对Volley的网络库工作流程做了进行了简单分析:

- [[Volley 网络库源码初探](http://wizardiy.com/2016/09/29/Volley%20网络库源码初探/)

Volley 中的图片加载方式总结起来有三种，分别为 ImageRequest、ImageLoader、NetworkImageView，这三种方式名称和使用方式各不相同，下面我们将通过每种加载方式的使用来对 Volley 的图片的加载框架进行分析。

### Volley 图片请求加载的三种方式

1. 使用ImageRequest方式加载,其请求示例代码如下:

        RequestQueue mQueue = Volley.newRequestQueue(context);
        ImageRequest imageRequest = new ImageRequest(url,
            new Response.Listener<Bitmap>() {  
                            @Override  
                            public void onResponse(Bitmap response) {  
                                imageView.setImageBitmap(response);  
                            }  
                        }, 0, 0, Config.RGB_565, new Response.ErrorListener() {  
                            @Override  
                            public void onErrorResponse(VolleyError error) {  
                                imageView.setImageResource(R.drawable.default_image);  
                            }  
                        });  
        mQueue.add(imageRequest);

2. 使用ImageLoader方式加载,其请求示例代码如下:

          RequestQueue mQueue = Volley.newRequestQueue(context);
          ImageLoader imageLoader = new ImageLoader(mQueue, new ImageCache() {  
              @Override  
              public void putBitmap(String url, Bitmap bitmap) {  
              }  

              @Override  
              public Bitmap getBitmap(String url) {  
                  return null;  
              }  
          });  
          ImageListener listener = ImageLoader.getImageListener(imageView, R.drawable.default_image, R.drawable.failed_image);
          imageLoader.get(url, listener);

3. 使用NetworkImageView方式加载,其请求示例代码如下:

        <com.android.volley.toolbox.NetworkImageView   
            android:id="@+id/network_image_view"  
            android:layout_width="200dp"  
            android:layout_height="200dp"  
            android:layout_gravity="center_horizontal"  
            />  

        RequestQueue mQueue = Volley.newRequestQueue(context);
        ImageLoader imageLoader = new ImageLoader(mQueue,ImageCache);
        networkImageView = (NetworkImageView) findViewById(R.id.network_image_view);  
        networkImageView.setDefaultImageResId(R.drawable.default_image);  
        networkImageView.setErrorImageResId(R.drawable.failed_image);  
        networkImageView.setImageUrl(url, imageLoader);  

### Volley ImageRequest 请求分析

方式1中ImageRequest的使用方式还是最常用的Request的使用方式,区别在于在deliverResponse的()结果不同(具体请求逻辑请参考文章头部的链接文章)。Volley的网络请求结果,如果缓存命中,则会调用如下方法对结果进行解析:

        // We have a cache hit; parse its data for delivery back to the request.
        request.addMarker("cache-hit");
        Response<?> response = request.parseNetworkResponse(
                new NetworkResponse(entry.data, entry.responseHeaders));
        request.addMarker("cache-hit-parsed");

如果缓存未命中,通过NetworkDispatcher进行网络请求,则会调用如下方法对结果进行解析:

        // Parse the response here on the worker thread.
        Response<?> response = request.parseNetworkResponse(networkResponse);
        request.addMarker("network-parse-complete");

不管是缓存命中(CacheDispatcher)还是不命中(NetworkDispatcher),都会通过`request.parseNetworkResponse(response)`的方式将网络结果的解析回调到Request中。则对于Request的子类实现ImageRequest,其解析方法实现如下代码所示:

        @Override
        protected Response<Bitmap> parseNetworkResponse(NetworkResponse response) {
            // Serialize all decode on a global lock to reduce concurrent heap usage.
            synchronized (sDecodeLock) {
                try {
                    return doParse(response);
                } catch (OutOfMemoryError e) {
                    VolleyLog.e("Caught OOM for %d byte image, url=%s", response.data.length, getUrl());
                    return Response.error(new ParseError(e));
                }
            }
        }

其中doParse()会进行对reponse的字节留进行处理,按请求参数包装成相应的Bitmap,其关键代码如下所示。

      byte[] data = response.data;
      BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
      Bitmap bitmap = null;
      if (mMaxWidth == 0 && mMaxHeight == 0) {
          decodeOptions.inPreferredConfig = mDecodeConfig;
          bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
      } else {
          // If we have to resize this image, first get the natural bounds.
          decodeOptions.inJustDecodeBounds = true;
          BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
          int actualWidth = decodeOptions.outWidth;
          int actualHeight = decodeOptions.outHeight;
          // Then compute the dimensions we would ideally like to decode to.
          int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,
                  actualWidth, actualHeight, mScaleType);
          int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,
                  actualHeight, actualWidth, mScaleType);
          // Decode to the nearest power of two scaling factor.
          decodeOptions.inJustDecodeBounds = false;
          // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
          // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
          decodeOptions.inSampleSize =
              findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
          Bitmap tempBitmap =
              BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

          // If necessary, scale down to the maximal acceptable size.
          if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth ||
                  tempBitmap.getHeight() > desiredHeight)) {
              bitmap = Bitmap.createScaledBitmap(tempBitmap,
                      desiredWidth, desiredHeight, true);
              tempBitmap.recycle();
          } else {
              bitmap = tempBitmap;
          }
      }
      if (bitmap == null) {
          return Response.error(new ParseError(response));
      } else {
          return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
      }

注意这段解析代码中maxWidth与maxHeight是用户初始化传入的请求参数,如果调用方不传的话,默认为0,不进行缩放。则解析的bitmap是原图的尺寸大小,而如果调用方传入参数,则需要对图片进行缩放(**PS:这段缩放代码其实也是很有参考意义的**)。在缩放完成后,将请求结果转入到Response对象中返回。之后,通过ResponseDelivery调用postResponse(reqeust, response)将结果回调到Listener的时候,通过deliverResponse回调的就是具体的解析结果了(从这里我们也可以看出,如果有自定义Request需要的时候,只需要继承Request,重写parseNetworkResponse与deliverResponse方法即可实现Volley中Request的自定义结果解析与回调)。

#####Volley ImageLoader 请求分析

ImageLoader中只有一个构造函数,它需要传入RequestQueue（我们从这点可以猜想ImageLoader可能也会依赖ImageReqeust进行请求）和ImageCache对象,而ImageCache是定义于ImageLoader类中的接口。
对于ImageCache说明如下源码所示:

      /**
        　Simple cache adapter interface. If provided to the ImageLoader, it
       　 will be used as an L1 cache before dispatch to Volley. Implementations
       　 must not block. Implementation with an LruCache is recommended.
      ＊/
      public interface ImageCache {
          public Bitmap getBitmap(String url);
          public void putBitmap(String url, Bitmap bitmap);
      }

这段代码的说明中注释说的很清楚，大意为ImageCache是一个简单的缓存适配接口,它提供给Volley作为一级缓存,其实现不能产生阻塞。推荐使用LruCache来实现这个接口。在上面的示例中,使用ImageCache直接返回了NULL,所以并没有起到缓存的效果。

ImageListener作为Imageloader中的一个接口,其继承了Response中的ErrorListener,从这个角度也可以看的出,Imageloader底层是通过Request执行的请求。而对于ImageListener的获取,如下代码所示:

        public static ImageListener getImageListener(final ImageView view,
                final int defaultImageResId, final int errorImageResId) {
            return new ImageListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    if (errorImageResId != 0) {
                        view.setImageResource(errorImageResId);
                    }
                }

                @Override
                public void onResponse(ImageContainer response, boolean isImmediate) {
                    if (response.getBitmap() != null) {
                        view.setImageBitmap(response.getBitmap());
                    } else if (defaultImageResId != 0) {
                        view.setImageResource(defaultImageResId);
                    }
                }
            };
        }

我们从默认实现也可以看的出,在response回调中会去Response中获取Bitmap,所以我们可以做一个推论:ImageLoader是通过ImageReqeust请求并对请求结果进行封装的一种图片加载方式。接下来我们对ImageLoader的加载过程进行详细分析。在为ImageLoader绑定注册监听回调后,我们可以使用ImageLoader的get方式加载图片。其中get有三种多态方法,如下所示:

      public ImageContainer get(String requestUrl, final ImageListener listener) {
          return get(requestUrl, listener, 0, 0);
      }

      public ImageContainer get(String requestUrl, ImageListener imageListener,
              int maxWidth, int maxHeight) {
          return get(requestUrl, imageListener, maxWidth, maxHeight, ScaleType.CENTER_INSIDE);
      }

      public ImageContainer get(String requestUrl, ImageListener imageListener,
              int maxWidth, int maxHeight, ScaleType scaleType) {
          ...
      }

我们可以看到,其中最终都是通过第三种方式进行获取,其中依赖传入参数数:url,listener,maxWidth,maxHeight,scaleType,如果不传ScaleType,默认的ScaleType为CenterInside。在ImageLoader通过get方式提交请求后，会首先到设置的一级缓存中查询图片缓存，如果缓存没有命中，则将request加入到RequestQueue中开始一个新请求，同时在mInFlightRequests中将请求加到请求队列中。源码如下：

        public ImageContainer get(String requestUrl, ImageListener imageListener,
                int maxWidth, int maxHeight, ScaleType scaleType) {

            // only fulfill requests that were initiated from the main thread.
            throwIfNotOnMainThread();

            final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);

            // Try to look up the request in the cache of remote images.
            Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
            if (cachedBitmap != null) {
                // Return the cached bitmap.
                ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
                imageListener.onResponse(container, true);
                return container;
            }

            // The bitmap did not exist in the cache, fetch it!
            ImageContainer imageContainer =
                    new ImageContainer(null, requestUrl, cacheKey, imageListener);

            // Update the caller to let them know that they should use the default bitmap.
            imageListener.onResponse(imageContainer, true);

            // Check to see if a request is already in-flight.
            BatchedImageRequest request = mInFlightRequests.get(cacheKey);
            if (request != null) {
                // If it is, add this request to the list of listeners.
                request.addContainer(imageContainer);
                return imageContainer;
            }

            // The request is not already in flight. Send the new request to the network and
            // track it.
            Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType,
                    cacheKey);

            mRequestQueue.add(newRequest);
            mInFlightRequests.put(cacheKey,
                    new BatchedImageRequest(newRequest, imageContainer));
            return imageContainer;
        }

**PS:这段代码会检查是否运行在主线程，如果运行在子线程中，则会直接抛出异常的。**
其中,ImageContainer是ImageLoader的内部类,在源码中的注释对它的介绍是：`Container object for all of the data surrounding an image request.`，意思是一个ImageRequest的数据对象包装类，它作为get方法的返回值，当有缓存的时候，它会将数据取出，直接回调同时返回包装类。而若不存在缓存，则先创建ImageContainer,回调listenert通知显示设置的默认图片。

接着这里会出现一个新的类型BatchedImageRequest，还有一个类型为HashMap的mInFlightRequests，mInFlightRequests是用来保存当前已存在的Request,若有相同请求时，它会将ImageContainer加入到BatchedImageRquest中，而BatchedImageRequest是ImageLoader的内部类，它是Request的包装类，其中有一个LinkedList去存储ImageContainer。

当有新的请求，并将请求请求加入到请求队列中的时候，就进入了Volley的网络请求逻辑中，这个我们前文已经分析过，这里就不再赘述。

在请求响应后（以响应成功为例），则会遍历所有的ImageContainer，回调到batchedRequest耦合的ImageContainer注册的ImageListener中。部分代码如下：

        protected void onGetImageSuccess(String cacheKey, Bitmap response) {
            // cache the image that was fetched.
            mCache.putBitmap(cacheKey, response);

            // remove the request from the list of in-flight requests.
            BatchedImageRequest request = mInFlightRequests.remove(cacheKey);

            if (request != null) {
                // Update the response bitmap.
                request.mResponseBitmap = response;

                // Send the batched response
                batchResponse(cacheKey, request);
            }
        }
        private void batchResponse(String cacheKey, BatchedImageRequest request) {
            mBatchedResponses.put(cacheKey, request);
            // If we don't already have a batch delivery runnable in flight, make a new one.
            // Note that this will be used to deliver responses to all callers in mBatchedResponses.
            if (mRunnable == null) {
                mRunnable = new Runnable() {
                    @Override
                    public void run() {
                        for (BatchedImageRequest bir : mBatchedResponses.values()) {
                            for (ImageContainer container : bir.mContainers) {
                                // If one of the callers in the batched request canceled the request
                                // after the response was received but before it was delivered,
                                // skip them.
                                if (container.mListener == null) {
                                    continue;
                                }
                                if (bir.getError() == null) {
                                    container.mBitmap = bir.mResponseBitmap;
                                    container.mListener.onResponse(container, false);
                                } else {
                                    container.mListener.onErrorResponse(bir.getError());
                                }
                            }
                        }
                        mBatchedResponses.clear();
                        mRunnable = null;
                    }

                };
                // Post the runnable.
                mHandler.postDelayed(mRunnable, mBatchResponseDelayMs);
            }
        }

在加载完成后，将缓存中的Request清空，这样整个请求就完成了。但这里还有一个问题，ImageLoader中默认构建的ImageListener的onResponse回调方法如下所示：

          public void onResponse(ImageContainer response, boolean isImmediate) {
                if (response.getBitmap() != null) {
                    view.setImageBitmap(response.getBitmap());
                } else if (defaultImageResId != 0) {
                    view.setImageResource(defaultImageResId);
                }
            }

**其中的isImmediate参数并未使用到，这个参数从其注释定义有这样的表述：isImmediate True if this was called during ImageLoader.get() variants.This can be used to differentiate between a cached image loading and a network　image loading in order to, for example, run an animation to fade in network loaded images．大意是指：这个参数如果是true,表明listener被调用是通过ImageLoader.get(),可以用来被区别图片是来自缓存还是网络加载。比如说在加载网络图片时来做一个淡入动画。但是我目前没看出这个参数有什么具体的实际意义，如果大家有什么想法可以在留言中提出。**

### Volley NetworkImageView 图片加载分析

对于第三种使用NetworkIMageView请求方式进行的图片加载,我们可以看到,在配置文件中配置后,却是使用ImageLoader进行加载的.在初始化NetworkImage后,设置好url就开始加载逻辑了。

        public void setImageUrl(String url, ImageLoader imageLoader) {
            mUrl = url;
            mImageLoader = imageLoader;
            // The URL has potentially changed. See if we need to load it.
            loadImageIfNecessary(false);
        }

调用loadImageIfNecessary()方法进行图片加载,而这个方法在NetworkImage中的onlayout中同样会调用代码如下所示(**我没想明白有什么场景是需要在在onLayout的时候去加载图片的╮（╯▽╰）╭**):

      protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
            super.onLayout(changed, left, top, right, bottom);
            loadImageIfNecessary(true);
      }

下面分析loadImageIfNecessary方法,我们可以看到如果传入url为空,则调用setDefaultImageOrNull()方法设置默认图片,否则的话才会继续加载图片。

     // if the URL to be loaded in this view is empty, cancel any old requests and clear the
    // currently loaded image.
    if (TextUtils.isEmpty(mUrl)) {
        if (mImageContainer != null) {
            mImageContainer.cancelRequest();
            mImageContainer = null;
        }
        setDefaultImageOrNull();
        return;
    }

假如有相同的图片在加载，则会直接返回。否则的话，会将之前的那个Request取消，重新加载现有请求。

    // if there was an old request in this view, check if it needs to be canceled.
    if (mImageContainer != null && mImageContainer.getRequestUrl() != null) {
        if (mImageContainer.getRequestUrl().equals(mUrl)) {
            // if the request is from the same URL, return.
            return;
        } else {
            // if there is a pre-existing request, cancel it if it's fetching a different URL.
            mImageContainer.cancelRequest();
            setDefaultImageOrNull();
        }
    }

最后，这些条件都不满足，则通过`mImageLoader.get()使用ImageLoader`进行请求加载，这样就进入了ImageLoader的加载逻辑。这里，我们也找到了上述疑惑的地方，在NetworkImageView中创建的ImageListener 回调中有如下逻辑：

    public void onResponse(final ImageContainer response, boolean isImmediate) {
        // If this was an immediate response that was delivered inside of a layout
       // pass do not set the image immediately as it will trigger a requestLayout
       // inside of a layout. Instead, defer setting the image by posting back to
       // the main thread.
      if (isImmediate && isInLayoutPass) {
          post(new Runnable() {
              @Override
              public void run() {
                   onResponse(response, false);
             　}
           });
           return;
     　}
       if (response.getBitmap() != null) {
                setImageBitmap(response.getBitmap());
       } else if (mDefaultImageId != 0) {
                setImageResource(mDefaultImageId);
        }
    }

**对于这段中的逻辑理解不了╮（╯▽╰）╭，有朋友有思路可以留言解惑。**

###结语
我们可以看出,ImageLoader对于图片逻辑的处理主要依赖于Request与RequestQueue框架,虽然整体使用较为繁琐，但是Volley对相关设置预留了扩展，总体来说如果使用Volley做网络库，但是又不想引入其他图片框架加大包体积的话，使用Volley来做图片加载也是一种不错的选择。

对于网络请求缓存的源码分析已经更新，详情点击：
- [源码学习｜Volley 网络请求缓存策略源码分析](http://www.jianshu.com/p/dd81b0e692c7)

*PS:整体层次图，仅供参考，如果不足，欢迎指出*Ｏ（∩＿∩）Ｏ哈！

![图片请求整体框架层次图](http://upload-images.jianshu.io/upload_images/1489253-bb68ac44e93c08e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
