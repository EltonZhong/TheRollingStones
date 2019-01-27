---
title: Kotlin协程Api详解
desc: CoroutineScope、 CoroutineContext详解
cover: /img/welcome-cover.jpg
tags:
  - Code
categories:
  - Code
date: 2019-1-27 22:35:34
---
本篇文章解析Kotlin协程的CoroutineScope， CoroutineContext及其继承类， 旨在探讨并理解kotlin的协程使用，以及对各个协程api细节整理。

关于协程跨平台的实现以及跨平台使用介绍，欢迎去[我的博客:编译之后--Kotlin 跨平台协程](https://therollingstones.cn/2018/12/16/code/Kotlin/)看看，或者去看看官方的视频&ppt，这也是我本篇文章的基础:
* Presentations and videos:
  * [Introduction to Coroutines](https://www.youtube.com/watch?v=_hfBv0a09Jc) (Roman Elizarov at KotlinConf 2017, [slides](https://www.slideshare.net/elizarov/introduction-to-coroutines-kotlinconf-2017))
  * [Deep dive into Coroutines](https://www.youtube.com/watch?v=YrrUCSi72E8) (Roman Elizarov at KotlinConf 2017, [slides](https://www.slideshare.net/elizarov/deep-dive-into-coroutines-on-jvm-kotlinconf-2017))
  * [Kotlin Coroutines in Practice](https://www.youtube.com/watch?v=a3agLJQ6vt8) (Roman Elizarov at KotlinConf 2018, [slides](https://www.slideshare.net/elizarov/kotlin-coroutines-in-practice-kotlinconf-2018))

这篇文章很久前就已经打好了草稿，但是由于最近忙没有时间写，再加上最近都没有在写Kotlin。今天仔细又读了一些文档、源码，为了整理一下自己的思路，将自己的个人理解写在这里。

本文的前提是读者了解协程基本使用，见[官方中文文档](https://www.kotlincn.net/docs/reference/coroutines/basics.html)

`注：Kotlin版本迭代日新月异，本篇文章难免过时或有疏漏，请读者注意。`

## CoroutineScope
什么是CoroutineScope?
  CoroutineScope可以理解为协程的`作用域`，可以管理其域内的所有协程。一个CoroutineScope可以有许多的子scope。

  创建子scope的方式有许多种，常见的有：
* 使用lauch, async 等builder创建一个新的子协程。协程(AbstractCoroutine)继承了 CoroutineScope，从父scope中继承了协程上下文(见下文CoroutineContext) 以及Job（见下文）
* 使用coroutineScope Api创建新scope:
`public suspend fun <R> coroutineScope(block: suspend CoroutineScope.() -> R): R ` 
这个api主要用于方便地创建一个子域，并且管理域中的所有子协程。注意这个方法只有在所有 block中创建的子协程全部执行完毕后，才会退出。如下示例：
  ```kotlin
        // print输出的结果顺序将会是 1， 2， 3， 4
        coroutineScope {
            delay(1000)
            println("1")
            launch { 
                delay(6000) 
                println("3")
            }
            println("2")
            return@coroutineScope
        }
        println("4")
  ```
  与之类似的还有supervisorScope,区别是supervisorScope 在子协程失败时不影响其他子协程，而coroutineScope是将异常抛出。
* 继承CoroutineScope.这也是比较推荐的做法，用于处理具有生命周期的对象。
 ```kotlin
class SomethingWithLifecycle : CoroutineScope {

        // 创建一个Job,并用这个job来管理你的SomethingWithLifecycle的所有子协程
        private val job = Job()
        override val coroutineContext: CoroutineContext
             get() = Dispatchers.Main + job

        // 当生命周期销毁时，结束所有子协程
        fun close() { job.cancel() }
}
```

## CoroutineContext
这里我将CoroutineContext翻译为协程上下文。 我也找不到更好的词去翻译它。协程上下文包含当前协程scope的信息， 比如的Job, ContinuationInterceptor, CoroutineName 和CoroutineId。在CoroutineContext中，是用map来存这些信息的， map的键是这些类的伴生对象，值是这些类的一个实例。 这个设计也是蛮骚的， 导致你可以这样子取得context的信息:
```kotlin
val job = context[Job]
val continuationInterceptor = context[ContinuationInterceptor]
```
 每一个信息(Element)的继承关系大概是这样的， 我拿Job来举例：
> Job.Key继承了CoroutineContext.Key, Job 继承了CoroutineContext.Element，而CoroutineContext.Element继承了 CoroutineContext。

name 和 id都无关紧要，我们仔细讲讲Job和ContinuationInterceptor。

### Job
如上文提到的，Job继承了CoroutineContext.Element， 他是协程上下文的一部分。 Job有两个重要的子类实现:JobSupport，提供Job 的父子关系管理， 和AbstractCoroutine，即协程。

Job对象持有所有的子job实例，可以取消所有子job的运行。Job的join方法会等待自己以及所有子job的执行， 所以Job给予了CoroutineScope一个管理自己所有子协程的能力。

#### JobSupport
JobSupport这边的api已经被标记为废弃了，将来可能会有变动。其实这边api设计我感觉有点看不懂，十分地confusing。以后看看有没有变化吧，这部分主要是提供一个 工厂方法 `Job()`，可以为你的组件提供一个初始的context， 一个JobImpl的实例。JobSupport同时是AbstractCoroutine的父类。

#### AbstractCoroutine
讲讲Job最重要的子类: AbstractCoroutine。这个类就可以直接理解为是一个协程对象了，他继承了CoroutineScope， Job， Continuation， JobSupport。

常见地， 使用launch 或者async方法都会实例化出一个AbstractCoroutine 的协程对象，而这个协程对象因为继承了CoroutineScope，所以拥有一个协程上下文。 一个协程的协程上下文的Job值就是他本身，即：
```kotlin
            val j = launch { delay(6000) }
            // True!
            println(j[Job] == j)
```

而关于Continuation，是kotlin内部用来跳转恢复协程用的。这个东西对外部暴露的东西不多，我们需要了解的有:
* suspendCoroutine, 挂起当前协程，利用Continuation继续协程的执行，常用于与其他平台阻塞方法兼容，这是1.1的api，不多做介绍
* SequenceBuilderIterator，这个就是kotlin协程的生成器了，详见[官方文档](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#generators)

### ContinuationInterceptor
这个接口主要是由各种dispather实现。 见[官方文档](https://www.kotlincn.net/docs/reference/coroutines/coroutine-context-and-dispatchers.html)
肚子好饿，不详细介绍了，后面特别补上一篇文章细说dispatcher在各个平台的表现吧， 敬请期待。

## The end
最后推荐个神仙仓库吧， [kotlin的各种提案](https://github.com/Kotlin/KEEP),
在这里面可以看到各种kotlin特性设计的方案。比如协程的:
> https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#generators

`如果觉得本篇文章对您有所帮助， 点个赞吧~~蟹蟹~`