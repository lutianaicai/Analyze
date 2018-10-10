# Kingfisher源码分析

## 缘起
最近解决公司项目缓存问题时候发现自己对沙盒理解还不深刻，就从项目中看了一下喵神的实现，惊叹于Kingfisher这个库的整洁规范。所以就萌生了写一篇文章记录阅读源码这件事情的意愿。

## 从接口谈起
首先还是要稍微介绍一下Kingfisher:
> Kingfisher is a lightweight, pure-Swift library for downloading and caching images from the web. This project is heavily inspired by the popular [SDWebImage](https://github.com/rs/SDWebImage). It provides you a chance to use a pure-Swift alternative in your next app.

喵神自己说受到了[SDWebImage](https://github.com/rs/SDWebImage)的启发，Kingfisher能给你一种纯Swift的体验。各位应该都不会陌生，就从这个最基本的方法开始分析吧：

```swift
let url = URL(string: "url_of_your_image")
imageView.kf.setImage(with: url)
```
相较于SDwebImage的

```objectivec
[self.imageView sd_setImageWithURL:[NSURL URLWithString:@"url"]
                  placeholderImage:[UIImage imageNamed:@"placeholder.png"]];

```
Kingfisher这里依旧保持了调用的简洁风格，主要功能一行代码。这里的方法调用体现了Swift跟OC的一点风格差异，在这里稍微展开谈谈。
###kf与sd_
`SDWebImage`的主要方法采取的是加前缀的方式进行对扩展方法的标记。
`Kingfisher`既然是纯Swift框架就必然从*OOP*（object-oriented programming）过渡到了*POP*（protocol oriented programming）我们来看`kf`这种链式调用是如何实现的

1.首先定义一个不可继承且具有`Base`范型的class

```swift
public final class Kingfisher<Base> {
    public let base: Base
    public init(_ base: Base) {
        self.base = base
    }
}
```

2.接着定义一个`KingfisherCompatible`协议，协议定义了一个只读属性`kf`。

```swift
public protocol KingfisherCompatible {
    associatedtype CompatibleType
    var kf: CompatibleType { get }
}
```

3.接着在扩展中实现`KingfisherCompatible`协议，调用`kf`返回一个`Kingfisher`对象。

```swift
public extension KingfisherCompatible {
    public var kf: Kingfisher<Self> {
        return Kingfisher(self)
    }
}
```

4.使`Kingfisher`中定义的`Image`，`ImageView`，`Button`遵循`KingfisherCompatible`。

```swift
extension Image: KingfisherCompatible { }
extension ImageView: KingfisherCompatible { }
extension Button: KingfisherCompatible { }
```

接着就可以通过限定扩展`extesion...where...`对不同的类添加特定的方法了。比如：

```swift
extension Kingfisher where Base: ImageView {
	...
}
```

##一探究竟
###ImageView+Kingfisher
我们就以[ImageView+Kingfisher](https://github.com/onevcat/Kingfisher/blob/master/Sources/ImageView%2BKingfisher.swift
      )为例

```swift
/**
     Set an image with a resource, a placeholder image, options, progress handler and completion handler.
     
     - parameter resource:          Resource object contains information such as `cacheKey` and `downloadURL`.
     - parameter placeholder:       A placeholder image when retrieving the image at URL.
     - parameter options:           A dictionary could control some behaviors. See `KingfisherOptionsInfo` for more.
     - parameter progressBlock:     Called when the image downloading progress gets updated.
     - parameter completionHandler: Called when the image retrieved and set.
     
     - returns: A task represents the retrieving process.
*/
    @discardableResult
    public func setImage(with resource: Resource?,
                         placeholder: Placeholder? = nil,
                         options: KingfisherOptionsInfo? = nil,
                         progressBlock: DownloadProgressBlock? = nil,
                         completionHandler: CompletionHandler? = nil) -> RetrieveImageTask 
```
这个就是给ImageView设置图片的主方法，因为Swift方法的参数可空导致这里就一个主方法而不像OC内种写一个万能方法再写一堆缺少某些参数的便利方法去调用万能方法。

其实喵神这里注释写的已经非常清楚了，我这里简单展开说明一下。

####资源Resource
参数第一个`resource`其实是一个协议，它定义了从缓存查找图片的`cacheKey`以及网络加载的`downloadURL`

```swift
public protocol Resource {
    /// The key used in cache.
    var cacheKey: String { get }
    
    /// The target image URL.
    var downloadURL: URL { get }
}
```

以及主方法实现的第一步就是对`resource`进行判空，如果为空就返回一个空`RetrieveImageTask`

```swift
guard let resource = resource else {
    self.placeholder = placeholder
    setWebURL(nil)
    completionHandler?(nil, nil, .none, nil)
    return .empty
}
```

如果不为空那么就给`ImageView+Kingfisher`动态添加的属性`webURL`赋值

```swift
setWebURL(resource.downloadURL)
```

####占位图
再说第二个参数`placeholder`，它也是一个协议（所谓从一个协议开始），很简单就是定义了两个方法，一个添加占位图，一个删除占位图

```swift
public protocol Placeholder {
    
    /// How the placeholder should be added to a given image view.
    func add(to imageView: ImageView)
    
    /// How the placeholder should be removed from a given image view.
    func remove(from imageView: ImageView)
}
```
在主方法中是这样添加占位图的

```swift
if !options.keepCurrentImageWhileLoading || noImageOrPlaceholderSet {
// Always set placeholder while there is no image/placeholder yet.
    self.placeholder = placeholder
}
```
注释写的清楚，当还没有image或placeholder时，总是设置一个placeholder

####策略options
第三个参数`options`这回不是协议了，它是一个包含不同策略信息的数组。诸如强制刷新啊，只内存缓存图片啊之类。如果用户不进行配置那么系统将使用默认

```swift
var options = KingfisherManager.shared.defaultOptions + (options ?? KingfisherEmptyOptionsInfo)
```

剩下两个参数分别是`progress`回调和`completion`回调就不细说了。

####获取图片
这些参数都准备好了就开始让`KingfisherManager.shared`调用下面方法去获取图片

```swift
func retrieveImage(with resource: Resource,
        options: KingfisherOptionsInfo?,
        progressBlock: DownloadProgressBlock?,
        completionHandler: CompletionHandler?) -> RetrieveImageTask
```

这里根据`options`是否强制刷新，如果强制刷新那么直接走网络下载并缓存的方法，否则先去缓存中获取图片然后在回调`completionHandler`里去添加图片

####safeAsync
回调里跟`SDWebImage`一样同样有个`safeAsync`

```swift
DispatchQueue.main.safeAsync {
    ...
    strongBase.image = image
    ...
}
```

这个`safeAsync`是个`DispatchQueue`的扩展

```swift
// This method will dispatch the `block` to self.
// If `self` is the main queue, and current thread is main thread, the block
// will be invoked immediately instead of being dispatched.
func safeAsync(_ block: @escaping ()->()) {
    if self === DispatchQueue.main && Thread.isMainThread {
        block()
    } else {
        async { block() }
    }
}
```
目的也很明确保证图像的绘制在主线程完成

###KingfisherManager
刚才提到了获取图片的方法就是`KingfisherManager`调用的，它是一个全局唯一的单例它协调`ImageDownloader`以及`ImageCache`来进行图片的获取。

####downloadAndCacheImage
首先说下载并缓存，如果在获取图片的`options`里策略选择了`forceRefresh`那么`KingfisherManager`会率先调用


```swift
func downloadAndCacheImage(with url: URL,
                             forKey key: String,
                      retrieveImageTask: RetrieveImageTask,
                          progressBlock: DownloadProgressBlock?,
                      completionHandler: CompletionHandler?,
                                options: KingfisherOptionsInfo) -> RetrieveImageDownloadTask?
```

在这个方法的回调里先是进行错误判断，如果`url`下载这边失败了就尝试从缓存中加载图片


```swift
if let error = error, error.code == KingfisherError.notModified.rawValue {
    // Not modified. Try to find the image from cache.
    // (The image should be in cache. It should be guaranteed by the framework users.)
    targetCache.retrieveImage(forKey: key, options: options, completionHandler: { (cacheImage, cacheType) -> Void in
        completionHandler?(cacheImage, nil, cacheType, url)
    })
    return
}
```

网络获取图片成功之后就会按照`options`调用下面方法

```swift
open func store(_ image: Image,
                      original: Data? = nil,
                      forKey key: String,
                      processorIdentifier identifier: String = "",
                      cacheSerializer serializer: CacheSerializer = DefaultCacheSerializer.default,
                      toDisk: Bool = true,
                      completionHandler: (() -> Void)? = nil)
```

按照喵神的注释
>Store an image to cache. It will be saved to both memory and disk. It is an async operation.

这是一个异步操作，没有什么更多要解释的。值得一提的是，这里储存参数有一个`original`这个是Data格式的，这里处理是为了控制磁盘的大小，喵神建议大家为了磁盘更好的储存性能尽量提供`original`。

####tryToRetrieveImageFromCach
如果没有强制要求更新，都是首先去缓存中查找图片的，也就是先走下面这个方法

```swift
func tryToRetrieveImageFromCache(forKey key: String,
                                       with url: URL,
                              retrieveImageTask: RetrieveImageTask,
                                  progressBlock: DownloadProgressBlock?,
                              completionHandler: CompletionHandler?,
                                        options: KingfisherOptionsInfo)
```

这个方法里定义了一个回调

```swift
let diskTaskCompletionHandler: CompletionHandler = { (image, error, cacheType, imageURL) -> Void in
            completionHandler?(image, error, cacheType, imageURL)
}
```

然后调用方法

```swift
func retrieveImage(forKey key: String,
                               options: KingfisherOptionsInfo?,
                     completionHandler: ((Image?, CacheType) -> Void)?) -> RetrieveImageDiskTask?
```
首先在缓存中查找这张图片如果找到了就直接闭包返回

```swift
if image != nil {
    diskTaskCompletionHandler(image, nil, cacheType, url)
    return
}
```


