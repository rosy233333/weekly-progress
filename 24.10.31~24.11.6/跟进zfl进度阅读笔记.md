# 跟进zfl进度阅读笔记

时间：2024/11/5

## Async Trait的讨论

Rust语言当前不支持包含async函数的trait（AFIT）的动态分发。[此处](https://github.com/zjp-CN/dyn-afit?tab=readme-ov-file)一些间接解决该问题的方案，各种方案的特点如下表。

![](../图片/屏幕截图%202024-11-05%20155000.png)

async-os中采用了方法4：

1. 定义`XX`和`AsyncXX`两个Trait，前者的函数返回值为`Poll<T>`，可以动态分发；后者包含async函数，不能动态分发。
2. 在`AsyncXX`中提供默认实现，并静态地让所有实现`XX`的类型均实现`AsyncXX`。
3. 使用时，根据`XX`来动态分发，再通过`AsyncXX`的默认实现使用`AsyncXX`中的async函数。

![](../图片/屏幕截图%202024-11-05%20161533.png)

该方法的优势是不需要额外的空间分配，因此具有比其它方法更好的性能。[zfl学长的测试结果](https://github.com/AsyncModules/async-os/discussions/3#discussioncomment-11139953)

但不足在于改变了对外提供的接口：原本的接口是一个async函数（即返回`Future`的函数），使用该方法后，接口变为了一个返回`Poll<T>`的函数。

## 统一同步异步接口

[学长的说明文档](https://github.com/AsyncModules/async-os/blob/dev/docs/unified-api.md)

简化版的代码如下：

```Rust
pub fun action(...) -> ActionFuture {
    #[cfg(feature = "thread")]
    {
        // 用于线程的动作代码
    }
    ActionFuture::new()
} 

impl Future for ActionFuture {
    type Output = ();
    fn poll(...) -> Poll<Self::Output>{
        #[cfg(not(feature = "thread"))]
        {
            // 用于协程的动作代码
        }
    }
}
```

于是，线程可以通过`action(...)`的方式调用接口，而协程可以通过`action(...).await`的方式调用接口。

学长也进行了更详细的设计，从而支持带返回值的接口。以及对不同调用方式（是否.await）、不同环境（async或非async）、`thread` feature是否启用的不同情况的组合都实现了支持。

不过，实现方式目前只与`thread` feature的启用有关，例如如果启用了`thread` feature，在async环境内调用`action(...).await`，也会使用线程的实现。

## trap和系统调用

## 文件系统