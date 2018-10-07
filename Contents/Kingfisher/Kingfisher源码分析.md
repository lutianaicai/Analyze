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
                         completionHandler: CompletionHandler? = nil) -> RetrieveImageTask {}
```
这个就是给ImageView设置图片的主方法，因为Swift方法的参数可空导致这里就一个主方法而不像OC内种写一个万能方法再写一堆缺少某些参数的便利方法去调用万能方法。

其实喵神这里注释写的已经非常清楚了，我这里简单展开说明一下。

参数第一个`Resource`其实是一个协议，它定义了从缓存查找图片的cacheKey以及网络加载的downloadURL

```swift
public protocol Resource {
    /// The key used in cache.
    var cacheKey: String { get }
    
    /// The target image URL.
    var downloadURL: URL { get }
}
```

以及方法实现的第一步就是对`resource`进行判空，如果为空就返回一个空`RetrieveImageTask`

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

再说第二个参数`Placeholder`，它也是一个协议（所谓从一个协议开始），很简单就是定义了两个方法，一个添加placeholder，一个删除placeholder

```swift
public protocol Placeholder {
    
    /// How the placeholder should be added to a given image view.
    func add(to imageView: ImageView)
    
    /// How the placeholder should be removed from a given image view.
    func remove(from imageView: ImageView)
}
```


