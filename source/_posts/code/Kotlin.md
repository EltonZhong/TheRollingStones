---
title: '编译之后--Kotlin 协程'
cover: imgs/lips.png
tags:
  - Code
categories:
  - code
desc: "Kotlin协程在跨平台项目的使用"
date: 2018-12-16 21:37:45
---

# Kotlin 协程
本文只是浅析 Kotlin 协程在各平台的实现, 以及跨平台兼容方案。 如果要看Kotlin协程api的用法，请移步[我的另一篇文章](https://therollingstones.cn/2018/12/25/code/kotlin/KotlinCoroutine/)

言归正传, 我们聊聊"协程".

***如果有讲的不好的地方， 欢迎在下方评论***

## 协程介绍
首先， 什么是协程？
>有人说协程拥有自己的寄存器上下文和栈的轻量级执行单元， 可以在控制执行上下文的切换。

>熟悉Javascript生成器的人可能会说，协程就是回调的语法糖，本质根本不关方法栈上下文切换啥事。

到底谁说的是对的？
别着急， 看看维基百科是怎么说的：

我们先看看下面这段描述:
>Coroutines are computer-program components that generalize subroutines for non-preemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations

协程就是可以生成非抢占式子程序的计算机程序， 可以允许程序有多个特定的地方可以挂起或者恢复执行。

## Kotlin 协程在各平台编译成什么？

>熟悉Javascript生成器的人可能会说， 协程就是回调的语法糖， 本质根本不关方法栈上下文切换啥事。

在Kotlin 协程之前， 我们讲讲关于Js的故事。
>大家都知道Js有诸多版本， 我们经常用着Es6、7、8的语法, 使用Babel将它们编译成低版本， 再在浏览器或者低版本node执行。 这里我们单纯对Node.js进行讨论。

>低版本的Node有协程吗？ 你可以说没有。实现 coroutine 的方式有很多，比如 ES6 的 generator，ES8(`你没有看错, 不是ES7`) 的 async/await。在低版本的Node中， 自然是没有协程的， 高级的生成器函数将会编译成普通的Js回调、状态机代码(`我们后面会讲到`)， 但是在Node.js 8之后的版本， node.js原生支持了协程， 实现了寄存器上下文和栈的切换。

>所以， 我们是不是可以说一段高版本Js代码如果经过编译成低版本代码后， 就不在拥有协程了呢？
> 这么说是不公平的, 如果这么说的话， 那本文就该换标题了， Kotlin就没有协程了
>>`上层不应该关心下层的具体实现，而只应该关心下层提供的接口`


大家都知道， Jvm也是没有原生协程的，大部分平台， 包括Kotlin-js编译后的es5 js代码， 也并没有关于协程的原生实现, (或者说没有用到)。 那么把协程看做重要特性的Kotlin， 编译后是怎么样的呢？
`Talk is cheap, show me your code.`
>让我们看看下面这段(js 和 kotlin 混写的)伪代码

```js
 suspend function postItems(item) {
     let tk = getToken(item)
     let post = doPost(tk)
     return post
 }

 interface StateMachine {
     status: number,
     item: Any,
     continuation: Continuation
 }

 function postI(item, sm: Continuation) {
     if (!sm instanceof StateMachine) {
         sm = {
             status: 0,
             item: null,
             continuation: sm
         }
     }

     switch sm.status:
         case 1:
             sm.status += 1;
             getToken(item, (tk) => {
                 sm.item = tk;
                 postI(null, sm)
             })
         case 2:
             sm.status += 1;
             doPost(sm.item, (post) => {
                 sm.item = post;
                 postI(null, sm)
             })
         case 3:
             sm.continuation.resume(sm.item)
             return
 }
```

**简析：**
> postItems 是一个kotlin的suspend方法， 他先通过http请求拿到token， 再通过token发起一个http post 请求， 将返回的对象 返回给调用者。 

>对应编译后的实现就成了postI,一个普通的方法,接受一个Continuation类型对象的方法。而这个 Continuation,可以近似看做一个回调函数, 也就是说, 在jvm这边, 你的suspend function, 最终变成了接受一个回调函数为参数的普通方法, 而且在其它平台也大抵类似。

>那么suspend function里面的suspend function调用是怎么调的呢？看上面的伪代码，你大概可以知道： 通过回调, 在getToken的回调里面继续调用postItems方法，但是状态已经改变，则可以执行到下一个suspend挂起点，以此类推,通过这个状态机实现了原来kotlin suspend 方法的`挂起与恢复`。

## Kotlin 跨平台实现
虽然Kotlin 编译后的普通方法提供了回调的应用， 但是正如官方文档中所说， 你永远不要尝试在各个平台实现这些回调， 除非你是个 `"coroutine master"`, 实现context等都是有坑的。

### Kotlin 协程 node.js
kotlin.coroutines 有个专门的 promise 库， 所以如果你要暴露一个api给外面的js库去调用， 你可以这么写： 
```kotlin
suspend fun helloWorld() {
  delay(1000)
  return "1212"
}

fun theApiYouWantToExposeAndUsedInOtherJsModules() {
  Globalscope.promise {
    helloworld()
  }
}
```

### Kotlin 协程 jvm
Jvm 有runblocking 的api阻塞当前线程执行， 这个是一个优点，因为在node.js单线程里面你没办法runblocking.
 所以你可以暴露这样的api: 
```kotlin
suspend fun helloWorld() {
  delay(1000)
  return "1212"
}

fun theApiYouWantToExposeAndUsedInOtherJvmProgram() = runBlocking {
    helloworld()
  }
}
```
当然为了性能更推荐的方式是利用jvm 独有future api 返回一个Future 对象:
```kotlin
fun theApiYouWantToExposeAndUsedInOtherJvmProgram() = future {
    helloworld()
  }
}
```

### Kotlin 协程 native

native 也有runblocking, 不过好像在swift 里面有个天坑。我对swift不是很了解， 解决方案给你扔这了： https://github.com/ktorio/ktor/issues/678

>如果有问题或者谬误， 欢迎在评论指出。