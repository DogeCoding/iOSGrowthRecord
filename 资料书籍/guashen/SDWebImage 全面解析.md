[TOC]

# `SDWebImage` 相关问题

## 1. `SDWebImage` 中使用 Associated Object 记录了什么？记录的值再后续的处理中又有什么用处？

在 `SDWebImage` 中的 `UIView+WebCache.m` 有以下源码：

```Objective-C
objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

在所有的 `UIView` 中记录了图片的 `url` 地址，并且这个 `url` 会当做缓存的 key 来存储。在写入文件的时候，会对其进行 MD5 加密保证安全性。

## 2. `SDWebImage` 中 `SDWebImageManager` 的作用和流程是什么？

`SDWebImageManager` 作为 Downloader 的一个入口，属于 API 层面到内部的桥梁。其中最重要的是是选定缓存策略和下载策略，在 `SDWebImage/SDWebImageManager.h` 文件中已经对这些枚举做出了定义。流程图如下所示：

![](http://7xwh85.com1.z0.glb.clouddn.com/manager.png)

## 3. `URLCallbacks` 这个字典的作用是什么？其结构怎样的？

用来存储 `URL` 的多个 `callback` 回调。因为加载一张图片可能会设置两个回调方法：`kProgressCallback` 和 `kCompletedCallback`，过程时和完成回调。其结构如下图：

![](http://7xwh85.com1.z0.glb.clouddn.com/URLCallBacks.png)

## 4. `SDWebImage` 是如何管理 `NSOperation` 来保证多个图片同时加载的？

在 Downloader 中，`SDWebImage` 通过构造一个 `NSOperation` 来实例化下载图片这个任务，并且会将其追加到 `NSOperationQueue *downloadQueue` 这个队列中。`SDWebImage` 还通过对于 `NSOperation` 的 `start` 方法的重写，实现了异步下载创建 `dataTask`、加锁保护、后台延时下载保活等操作，并在主线程中通知下载任务开始。

## 5. `SDWebImage` 用了哪个缓存机制？

在 `SDWebImage` 用到了 Memory 和 Disk 的双缓存。