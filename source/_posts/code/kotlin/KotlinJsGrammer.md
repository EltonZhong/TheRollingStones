---
title: Kotlin-js features.
cover: imgs/lips.png
tags:
  - Code
categories:
  - code
desc: the Official document is not detailed. I will talk about something more in this document, and why kotlin.js is better than typescript.
date: 2018-12-18 21:37:45
mp3: http://link.hhtjim.com/163/26609785.mp3
---

# About kotlin-multiplatform, Kotlin.js
After [Setup](https://therollingstones.cn/2018/12/18/code/kotlin/KotlinMultiPlatformSetup), we can start to know how to write kotlin-js code.

Actually, the document you need is [here](https://www.kotlincn.net/docs/reference/dynamic-type.html), but I will talk about something more in this document, and why kotlin.js is better than typescript.

## Compile
When kotlin code compiles to js, the type checking in function parameter is lost, just like typescript.
### Type checking in function
```kotlin
    fun hello(a: String) {

    }
```
Kotlin code above will compile to js below
```js
   function hello(a) {
        process[a];
   }
```
When you call hello() function, you can not call hello with an integer, but after  being compiled to js, you can do this, like below:
```js
   function hello(a) {
        process[a];
   }
   // All code below is OK in js.
   hello(1)
   hello({})
```

### While, the type is not lost(`Why kotlin is greater than typescript`)
Classes in kotlin will remain, and they will be used in some kotlin grammer, especially in serilization libs. So `Don't use simple objects instead of class object, it's different from typescipt`.

![Example](https://therollingstones.cn/imgs/AccountLockDto.png)

Code may explain better:
> You are Given a class `A` with a field `a`
```kotlin
interface A {
    val a: String
}

class B(override val a: String): A

fun helloA(a: A) {
    println(a)
    println(a::class)
}
```
> When you compile it to js:
```js
function A() {
}
A.$metadata$ = {
    kind: Kind_INTERFACE,
    simpleName: 'A',
    interfaces: []
};
function helloA(a) {
    println(a);
    println(Kotlin.getKClassFromExpression(a));
}

// And class B:
function B(a) {
    this.a_175lx$_0 = a;
}
Object.defineProperty(B.prototype, 'a', {
    get: function () {
    return this.a_175lx$_0;
    }
});
B.$metadata$ = {
    kind: Kind_CLASS,
    simpleName: 'B',
    interfaces: [A]
};

// Quite familar right? We can always write code below in typescript.
// But You'll get into trouble when you use this in kotlin-js!
// This will output 'class Any' for `::class call` call.
hello({a: ""})

// The prefered way is to use class B:
hello(new B(""))
```
> When typescript compiled to js, all the types are erased. 

> But in kotlin, it's not.

## Error handling:
You can not catch the specified errors in typescript(`What a shame!`) and js. but you can do it in kotlin.

### You can write multi Error catching blocks.

>Example:
```java
try {
    yourFunc()
} catch(e: Exception) {
   console.log(e);
} catch(e: Throwable) {
    console.log(e);
}
```
> Kotlin common Code below will be compiled to js code:

```js
try {
    yourFunc();
} catch (e) {
    if (Kotlin.isType(e, Exception)) {
        console.log(e);
    }
    else if (Kotlin.isType(e, Throwable)) {
        console.log(e);
    }
} else
    throw e;
}
```

### If you want to catch js error in kotlin:
Try code below
```java
try { ... } catch(e: dynamic) {
    // handle this error
}
```

## Coroutine

The coroutines in js are just the same as kotlin-common. While you can not call suspend function in js.

So you can transfer api to typical promise call:
```
suspend fun hello() {
    delay(1000)
}

fun helloApi() = promise {
    hello()
}
```

For more information, see [here](https://therollingstones.cn/2018/12/16/code/Kotlin/) to get further explain for kotlin Coroutine.