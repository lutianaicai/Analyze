# PromiseKit源码分析

面对回调地狱[PromiseKit](https://github.com/mxcl/PromiseKit)提供了一种简洁易用的异步编程模式。让你可以编写出更加易读，更加专注结果的代码。本文意在探寻简介背后的逻辑。

## 从最小类型谈去
搞清复杂封装的源码中最基本的类型就好比搞清一篇英文文章中所有看不懂的生词的意思。再想看懂文章，只需要把已知信息串联起来就行了。

### Box
`box`是`PromiseKit`中基本的类，在介绍它之前，先得说一下同样定义在Box.swift中的另外两个更小的类型`Sealant`和`Handlers`

```Swift
enum Sealant<R> {
    case pending(Handlers<R>)
    case resolved(R)
}

final class Handlers<R> {
    var bodies: [(R) -> Void] = []
    func append(_ item: @escaping(R) -> Void) { bodies.append(item) }
}
```

`Handlers<R>`用于保存`Promise`的结果,`bodies`属性记录了所有可能会接受`Promise`返回值的方法，例如 `.then` `.done`。
`Sealant`则表征Box的状态，`.pending`表示还在等待结果，`.resolved`表示已经接受了结果

说完这两个类型，下面进入正题，开始介绍`Box`

#### 基类Box

首先从`Box`基类讲起

```Swift
class Box<T> {
  func inspect() -> Sealant<T> { fatalError() }
  func inspect(_: (Sealant<T>) -> Void) { fatalError() }
  func seal(_: T) {}
}
```

它接受一个范型参数，约束了三个方法，但是其实这三个方法都近乎空实现

```Swift
func inspect() -> Sealant<T> { fatalError() }
```

我们根据返回值可以查询目前`box`的状态，是`.pending`还是`.resolved`

```Swift
func inspect(_: (Sealant<T>) -> Void) { fatalError() }
```

则是根据状态的不同选择不同的逻辑

```Swift
func seal(_: T) {}
```

是把值放入箱子然后封上。具体实现我们还得看`Box`的两个派生类`SealedBox`和`EmptyBox`

#### SealedBox

翻译过来就是`封上了的箱子`，看代码

```Swift
final class SealedBox<T>: Box<T> {
  let value: T

  init(value: T) {
    self.value = value
  }

  override func inspect() -> Sealant<T> {
    return .resolved(value)
  }
}
```

可以看出，创建封好的箱子时就要往箱子里放入一个`Promise`的结果，而且放过以后`Box`的状态就为`.resolved`了且值也不能修改了。

#### EmptyBox

空箱`EmptyBox`是`Box`类中最复杂也是最重要的派生类，先看属性

```Swift
private var sealant = Sealant<T>.pending(.init())
private let barrier = DispatchQueue(label: "org.promisekit.barrier", attributes: .concurrent)
```

其中：
* `sealant`表示`Box`的状态，初始值是`.pending`，关联一个空的`Handler`
* `barrier`是一个异步队列。内部通过这个队列尽心状态与读写的同步。

接下来看获取箱子状态的`inspect`方法

```Swift
override func inspect() -> Sealant<T> {
        var rv: Sealant<T>!
        barrier.sync {
            rv = self.sealant
        }
        return rv
    }
```

很简单，`barrier`中获取这个`sealant`并返回

再看设置状态的`inspect`方法

```Swift
override func inspect(_ body: (Sealant<T>) -> Void) {
        var sealed = false
        barrier.sync(flags: .barrier) {
            switch sealant {
            case .pending:
                // body will append to handlers, so we must stay barrier’d
                body(sealant)
            case .resolved:
                sealed = true
            }
        }
        if sealed {
            // we do this outside the barrier to prevent potential deadlocks
            // it's safe because we never transition away from this state
            body(sealant)
        }
    }
```

它同步判断了箱子的状态，如果是`.pending`就调用`body`参数，如果是`.resolved`就把箱子的密封状态设置成`true`然后再屌用`body`参数

最后再看封箱方法`seal`

```Swift
override func seal(_ value: T) {
        var handlers: Handlers<T>!
        barrier.sync(flags: .barrier) {
            guard case .pending(let _handlers) = self.sealant else {
                return  // already fulfilled!
            }
            handlers = _handlers
            self.sealant = .resolved(value)
        }

        if let handlers = handlers {
            handlers.bodies.forEach{ $0(value) }
        }
    }
```

首先在`barries`中同步判断，箱子如果封上了就直接顺序调用所有`handler`，如果还是`.pending`状态就修改状态至`.resolved`并且修改箱子里的值为`value`,然后再顺序调用所有的`handler`

### Resolver



