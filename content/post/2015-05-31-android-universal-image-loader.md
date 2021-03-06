---
title: "Android Universal Image Loader"
date: "2015-05-31"
tags: ["android"]
---
本文总结了著名的Android UIL图像加载和显示库的使用 <!--more-->

最近在阅读[Coding的安卓客户端源码](https://coding.net/u/coding/p/Coding-Android/git)，因为该源码的图片加载库使用的是[universal-image-loader](https://github.com/nostra13/Android-Universal-Image-Loader)，我以前也使用过，但是没总结过，所以这次好好研究并总结下它的使用方法。其实，这些类库使用起来不会很难，但是很多时候如果之前没有仔细阅读这些类库的相关文档，开发过程中由于时间紧迫常常会因为快速实现功能而没有采用官方推荐的最佳实践，这样对于应用来说其实是不好的。

#### 1.Android Universal Image Loader简介

Android Universal Image Loader简称UIL，是Android开源类库中声名远扬的一个，它的主要功能是异步图像加载，缓存和显示。你会感觉自己写一个其实也不难对吧，但是如果考虑周全，考虑多线程、缓存、内存溢出等很多方面的问题的话，要写一个通用的异步图片加载类库还是不简单的。

UIL aims to provide a powerful, flexible and highly customizable instrument for image loading, caching and displaying. It provides a lot of configuration options and good control over the image loading and caching process.

它的功能有：

1.Multithread image loading (async or sync)

多线程的图片加载 (同步或者异步)

2.Wide customization of ImageLoader's configuration (thread executors, downloader, decoder, memory and disk cache, display image options, etc.)

提供各种图片加载的参数配置

3.Many customization options for every display image call (stub images, caching switch, decoding options, Bitmap processing and displaying, etc.)

提供各种图片显示的参数配置

4.Image caching in memory and/or on disk (device's file system or SD card)

提供图片的内存和外存的缓存

5.Listening loading process (including downloading progress)

可以监听图片加载的进度

UIL的任务流程图

![image](/images/UIL_Flow.png)

#### 2.导入方式

Gradle

```java
compile 'com.nostra13.universalimageloader:universal-image-loader:1.9.4'
```

#### 3.使用方式

##### 3.1 图片加载

使用UIL之前需要设置一次参数，这个参数的设置是全局的，所以只需要设置一次，一般大家都是在`Application`类中进行初始化设置的。`ImageLoaderConfiguration`参数的含义从字面上都还是比较容易理解的，在[这里](https://github.com/nostra13/Android-Universal-Image-Loader/blob/master/library/src/com/nostra13/universalimageloader/core/ImageLoaderConfiguration.java#L199-599)可以查看所有可以设置的参数。也可以直接使用默认的参数配置，只需要调用`ImageLoaderConfiguration.createDefault(this);`即可。

```
// DON'T COPY THIS CODE TO YOUR PROJECT! This is just example of ALL options using.
// See the sample project how to use ImageLoader correctly.
File cacheDir = StorageUtils.getCacheDirectory(context);
ImageLoaderConfiguration config = new ImageLoaderConfiguration.Builder(context)
        .memoryCacheExtraOptions(480, 800) // default = device screen dimensions
        .diskCacheExtraOptions(480, 800, null)
        .taskExecutor(...)
        .taskExecutorForCachedImages(...)
        .threadPoolSize(3) // default
        .threadPriority(Thread.NORM_PRIORITY - 2) // default
        .tasksProcessingOrder(QueueProcessingType.FIFO) // default
        .denyCacheImageMultipleSizesInMemory()
        .memoryCache(new LruMemoryCache(2 * 1024 * 1024))
        .memoryCacheSize(2 * 1024 * 1024)
        .memoryCacheSizePercentage(13) // default
        .diskCache(new UnlimitedDiscCache(cacheDir)) // default 也可以自定义缓存路径
        .diskCacheSize(50 * 1024 * 1024)
        .diskCacheFileCount(100)
        .diskCacheFileNameGenerator(new HashCodeFileNameGenerator()) // default
        .imageDownloader(new BaseImageDownloader(context)) // default
        .imageDecoder(new BaseImageDecoder()) // default
        .defaultDisplayImageOptions(DisplayImageOptions.createSimple()) // default
        .writeDebugLogs()
        .build();
```

部分重要参数的含义 [内容来源](http://blog.csdn.net/jjwwmlp456/article/details/40788583)

```
	final Resources resources;
	final int maxImageWidthForMemoryCache;
	内存缓存图片的最大宽度
	final int maxImageHeightForMemoryCache;
	内存缓存图片的最大高度
	final int maxImageWidthForDiskCache;
	磁盘缓存图片的宽度
	final int maxImageHeightForDiskCache;
	磁盘缓存图片的高度
	final BitmapProcessor processorForDiskCache;
	磁盘缓存图片处理器
	final Executor taskExecutor;

	final Executor taskExecutorForCachedImages;
	缓存任务线程池
	final boolean customExecutor;

	final boolean customExecutorForCachedImages;
	自定义缓存线程池
	final int threadPoolSize;
	池里核心线程数
	final int threadPriority;
	线程优先级
	final QueueProcessingType tasksProcessingType;
	任务队列的类型：enum{LIFO、FIFO}
	final MemoryCache memoryCache;
	内存缓存
	final DiskCache diskCache;
	磁盘缓存
	final ImageDownloader downloader;
	图片下载器
	final ImageDecoder decoder;
	图片解析器
	final DisplayImageOptions defaultDisplayImageOptions;
    图片显示参数
	final ImageDownloader networkDeniedDownloader;
	图片下载器：禁止从网络加载
	final ImageDownloader slowNetworkDownloader;
	图片下载器：慢速网络加载
```

下面是UIL的Sample程序中的Application示例，供参考使用

```
import android.annotation.TargetApi;
import android.app.Application;
import android.content.Context;
import android.os.Build;
import android.os.StrictMode;
import com.nostra13.universalimageloader.cache.disc.naming.Md5FileNameGenerator;
import com.nostra13.universalimageloader.core.ImageLoader;
import com.nostra13.universalimageloader.core.ImageLoaderConfiguration;
import com.nostra13.universalimageloader.core.assist.QueueProcessingType;

/**
 * @author Sergey Tarasevich (nostra13[at]gmail[dot]com)
 */
public class UILApplication extends Application {
	@TargetApi(Build.VERSION_CODES.GINGERBREAD)
	@SuppressWarnings("unused")
	@Override
	public void onCreate() {
		if (Constants.Config.DEVELOPER_MODE && Build.VERSION.SDK_INT >= Build.VERSION_CODES.GINGERBREAD) {
			StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyDialog().build());
			StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder().detectAll().penaltyDeath().build());
		}

		super.onCreate();

		initImageLoader(getApplicationContext());
	}

	public static void initImageLoader(Context context) {
		// This configuration tuning is custom. You can tune every option, you may tune some of them,
		// or you can create default configuration by
		//  ImageLoaderConfiguration.createDefault(this);
		// method.
		ImageLoaderConfiguration.Builder config = new ImageLoaderConfiguration.Builder(context);
		config.threadPriority(Thread.NORM_PRIORITY - 2);
		config.denyCacheImageMultipleSizesInMemory();
		config.diskCacheFileNameGenerator(new Md5FileNameGenerator());
		config.diskCacheSize(50 * 1024 * 1024); // 50 MiB
		config.tasksProcessingOrder(QueueProcessingType.LIFO);
		config.writeDebugLogs(); // Remove for release app

		// Initialize ImageLoader with configuration.
		ImageLoader.getInstance().init(config.build());
	}
}
```

##### 3.2 图片显示

**简单描述一下图片的加载显示过程：每一个图片的加载和显示任务都运行在独立的线程中，除非这个图片缓存在内存中，这种情况下图片会立即显示。如果需要的图片缓存在本地，他们会开启一个独立的线程队列。如果在缓存中没有正确的图片，任务线程会从线程池中获取，因此，快速显示缓存图片时不会有明显的障碍。** [内容来源](http://blog.csdn.net/vipzjyno1/article/details/23206387)

图片显示的使用方法

```
//先得到单例实体imageLoader
ImageLoader imageLoader = ImageLoader.getInstance(); // Get singleton instance

// Load image, decode it to Bitmap and display Bitmap in ImageView (or any other view
//  which implements ImageAware interface)
//调用imageLoader.displayImage将imageUri对应的图片显示在imageView组件上
imageLoader.displayImage(imageUri, imageView);

// Load image, decode it to Bitmap and return Bitmap to callback
//监听图片加载，处理回调函数
imageLoader.loadImage(imageUri, new SimpleImageLoadingListener() {
    @Override
    public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
        // Do whatever you want with Bitmap
    }
});

// Load image, decode it to Bitmap and return Bitmap synchronously
//加载图片并将其解码成Bitmap
Bitmap bmp = imageLoader.loadImageSync(imageUri);
```

上面的几个方法的完整版本是

```
// Load image, decode it to Bitmap and display Bitmap in ImageView (or any other view
//  which implements ImageAware interface)
imageLoader.displayImage(imageUri, imageView, options, new ImageLoadingListener() {
    @Override
    public void onLoadingStarted(String imageUri, View view) {
        ...
    }
    @Override
    public void onLoadingFailed(String imageUri, View view, FailReason failReason) {
        ...
    }
    @Override
    public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
        ...
    }
    @Override
    public void onLoadingCancelled(String imageUri, View view) {
        ...
    }
}, new ImageLoadingProgressListener() {
    @Override
    public void onProgressUpdate(String imageUri, View view, int current, int total) {
        ...
    }
});
// Load image, decode it to Bitmap and return Bitmap to callback
ImageSize targetSize = new ImageSize(80, 50); // result Bitmap will be fit to this size
imageLoader.loadImage(imageUri, targetSize, options, new SimpleImageLoadingListener() {
    @Override
    public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
        // Do whatever you want with Bitmap
    }
});
// Load image, decode it to Bitmap and return Bitmap synchronously
ImageSize targetSize = new ImageSize(80, 50); // result Bitmap will be fit to this size
Bitmap bmp = imageLoader.loadImageSync(imageUri, targetSize, options);
```

其中涉及到了图片显示的参数设置，这是通过`DisplayImageOptions`来实现的，如果没有设置`DisplayImageOptions`的话就会使用默认的配置参数`ImageLoaderConfiguration.defaultDisplayImageOptions(...)`。

```
// DON'T COPY THIS CODE TO YOUR PROJECT! This is just example of ALL options using.
// See the sample project how to use ImageLoader correctly.
DisplayImageOptions options = new DisplayImageOptions.Builder()
        .showImageOnLoading(R.drawable.ic_stub) // resource or drawable
        .showImageForEmptyUri(R.drawable.ic_empty) // resource or drawable
        .showImageOnFail(R.drawable.ic_error) // resource or drawable
        .resetViewBeforeLoading(false)  // default
        .delayBeforeLoading(1000)
        .cacheInMemory(false) // default
        .cacheOnDisk(false) // default
        .preProcessor(...)
        .postProcessor(...)
        .extraForDownloader(...)
        .considerExifParams(false) // default
        .imageScaleType(ImageScaleType.IN_SAMPLE_POWER_OF_2) // default
        .bitmapConfig(Bitmap.Config.ARGB_8888) // default
        .decodingOptions(...)
        .displayer(new SimpleBitmapDisplayer()) // default
        .handler(new Handler()) // default
        .build();
```

图片显示的参数设置

```
	private final int imageResOnLoading;
	正在加载时显示的图片资源		id
	private final int imageResForEmptyUri;
	图片uri为空时显示的图片		id
	private final int imageResOnFail;		
	图片加载失败显示的图片		id
	private final Drawable imageOnLoading;
	正在加载时显示的图片资源		drawable
	private final Drawable imageForEmptyUri;
	图片uri为空时显示的图片		drawable
	private final Drawable imageOnFail;
	图片加载失败显示的图片		drawable
	private final boolean resetViewBeforeLoading;
	加载前是否重置view
	private final boolean cacheInMemory;
	是否启用内存缓存
	private final boolean cacheOnDisk;
	是否启用磁盘缓存
	private final ImageScaleType imageScaleType;
	图片缩放类型
	private final Options decodingOptions;
	BitmapFactory用到的options
	private final int delayBeforeLoading;
	延迟多久加载
	private final boolean considerExifParams;
	exif参数是否可用
	private final Object extraForDownloader;
	额外的下载对象
	private final BitmapProcessor preProcessor;
	bitmap加载前的处理
	private final BitmapProcessor postProcessor;
	bitmap加载时的处理
	private final BitmapDisplayer displayer;
	bitmap显示
	private final Handler handler;
	private final boolean isSyncLoading;
	是否同步加载图片(一个接一个的加载)
```

图片显示支持多种图片URI的格式

```
"http://site.com/image.png" // from Web
"file:///mnt/sdcard/image.png" // from SD card
"file:///mnt/sdcard/video.mp4" // from SD card (video thumbnail)
"content://media/external/images/media/13" // from content provider
"content://media/external/video/media/13" // from content provider (video thumbnail)
"assets://image.png" // from assets
"drawable://" + R.drawable.img // from drawables (non-9patch images)
```

Use `drawable://` only if you really need it! Always consider the native way to load drawables - `ImageView.setImageResource(...)` instead of using of `ImageLoader`.

**注意，如果是显示`drawable://`中的图片，最好还是使用`ImageView.setImageResource(...)`的方式，而不要使用`ImageLoader`**

#### 4. 其他特性

#####4.1 DiskCache 和 MemoryCache

DiskCache 和 MemoryCache都有一些默认的实现类

(1)interface `DiscCache` -> abstract `BaseDiscCache`

`UnlimitedDiscCache`            
(The **fastest** cache, doesn't limit cache size) - **Used by default**

`LimitedAgeDiscCache`           
(Size-unlimited cache with limited files' lifetime. If age of cached file exceeds defined limit then it will be deleted from cache.)

`LruDiscCache`            
(Cache limited by total cache size and/or by file count. If cache size exceeds specified limit then least-recently used file will be deleted)

(2)interface `MemoryCache` -> abstract `BaseMemoryCache`

(2.1) 只保存强引用的Cache策略

`LruMemoryCache`      
(Least recently used bitmap is deleted when cache size limit is exceeded) - **Used by default**

(2.2) 只保存弱引用的Cache策略

`WeakMemoryCache`
(Unlimited cache)

(2.3) 只保存强和弱引用的Cache策略

`LimitedAgeMemoryCache`      
(Decorator. Cached object is deleted when its age exceeds defined value)

`LRULimitedMemoryCache`     
(Least recently used bitmap is deleted when cache size limit is exceeded)

`FIFOLimitedMemoryCache`      
(FIFO rule is used for deletion when cache size limit is exceeded)     

`UsingFreqLimitedMemoryCache`      
(Least frequently used bitmap is deleted when cache size limit is exceeded)

**Caching is NOT enabled by default**

默认情况下，图片是并不会缓存的！只有显式配置了缓存才会对图片进行缓存的！配置方式有两种：

```
// Create default options which will be used for every
//  displayImage(...) call if no options will be passed to this method
DisplayImageOptions defaultOptions = new DisplayImageOptions.Builder()
        ...
        .cacheInMemory(true)
        .cacheOnDisk(true)
        ...
        .build();
ImageLoaderConfiguration config = new ImageLoaderConfiguration.Builder(getApplicationContext())
        ...
        .defaultDisplayImageOptions(defaultOptions)
        ...
        .build();
ImageLoader.getInstance().init(config); // Do it on Application start

// Then later, when you want to display image
ImageLoader.getInstance().displayImage(imageUrl, imageView); // Default options will be used
```

或者

```
DisplayImageOptions options = new DisplayImageOptions.Builder()
        ...
        .cacheInMemory(true)
        .cacheOnDisk(true)
        ...
        .build();
ImageLoader.getInstance().displayImage(imageUrl, imageView, options); // Incoming options will be used
```

##### 4.2 BitmapDisplayer

`BitmapDisplayer`接口有一个抽象的方法`void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom);`。关于`ImageAware`，调用时传`new ImageViewAware(imgview)`就可以。

`FadeInBitmapDisplayer`           
淡入动画显示图片            
`RoundedBitmapDisplayer`          
圆角显示图片            
`RoundedVignetteBitmapDisplayer`  
圆角显示图片，颜色会有个渐变的效果         
`SimpleBitmapDisplayer`           
简单实现          

##### 4.3 FileNameGenerator

图片文件名生成方式

`HashCodeFileNameGenerator`   
以imguri的hashcode为name      
`Md5FileNameGenerator`        
以imguri的md5值为name        

##### 4.4 ImageScaleType

图片的缩放类型`ImageScaleType`:

`EXACTLY`:图像将完全按比例缩小的目标大小     
`EXACTLY_STRETCHED`:图片会缩放到目标大小      
`IN_SAMPLE_INT`:图像将被二次采样的整数倍      
`IN_SAMPLE_POWER_OF_2`:图片将降低2倍，直到下一减少步骤，使图像小于目标大小      
`NONE`:图片不会调整       

##### 4.5 UIL是如何计算所需的图片的大小的呢？

How UIL define Bitmap size needed for exact ImageView? It searches defined parameters:

Get actual measured `width` and `height` of ImageView       
Get `android:layout_width` and `android:layout_height` parameters      
Get `android:maxWidth` and/or `android:maxHeight` parameters     
Get maximum width and/or height parameters from configuration (`memoryCacheExtraOptions(int, int)` option)       
Get `width` and/or `height` of device screen      

So try to set `android:layout_width|android:layout_height` or `android:maxWidth|android:maxHeight` parameters for ImageView if you know approximate maximum size of it. It will help correctly compute Bitmap size needed for this view and **save memory**.

UIL会根据上面的数据来计算要显示在该ImageView组件上的图片的大小，所以，如果可以的话尽量设置好图片的`android:layout_width|android:layout_height` 或者 `android:maxWidth|android:maxHeight`参数信息，这样可以方便UIL正确计算Bitmap的大小以便节省空间。

##### 4.6 OOM

内存溢出是使用UIL比较容易出现的问题，如果出现这个问题的话，首先考虑禁用部分的Cache in Memory，如果还是不行，可以考虑下面几点([内容来源](http://blog.csdn.net/vipzjyno1/article/details/23206387)+[Useful info](https://github.com/nostra13/Android-Universal-Image-Loader/wiki/Useful-Info))：

①减少配置之中线程池的大小(`threadPoolSize`)，推荐1-5个线程即可    

②尽量使用`bitmapConfig(Bitmap.config.RGB_565)`代替`ARGB_8888`，前者更加节省内存      

③尽量使用`imageScaleType(ImageScaleType.EXACTLY)`        

④避免使用`RoundedBitmapDisplayer`，它会创建新的`ARGB_8888`格式的`Bitmap`对象      

⑤在Configuration中使用`diskCacheExtraOptions(480, 320, null)`

⑥使用`memoryCache(new WeakMemoryCache())`，不要使用`cacheInMemory()`    

##### 4.7 Image URL

在打印的日志中我们常常可以看到这样格式的图片网址：`http://anysite.com/images/image.png_230x460`，这并不是UIL实际请求的图片网址，后面的数字是`targetSize`，前面才是实际的图片网址，注意这点。

OK！有上面这些内容足够我们日常开发灵活使用UIL啦，Enjoy it！
