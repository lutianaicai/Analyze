# PromiseKit源码分析

面对回调地狱[PromiseKit](https://github.com/mxcl/PromiseKit)提供了一种简洁易用的异步编程模式。让你可以编写出更加易读，更加专注结果的代码。本文意在探寻简介背后的逻辑。

## 从最小类型谈去
搞清复杂封装的源码中最基本的类型就好比搞清一篇英文文章中所有看不懂的生词的意思。再想看懂文章，只需要把已知信息串联起来就行了。

### BOX
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
`Handlers<R>`用于保存`Promise`的结果



### Resolver



