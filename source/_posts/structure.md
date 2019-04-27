---
title: 测试项目结构
desc: 怎么样的设计才算是好的设计？
cover: imgs/lips.png
tags:
  - Code
categories:
  - Code
date: 2019-03-03 21:15:31
---

## 一点思考

> Last updated at 2019.4.27

这么晚才来把这篇文章补全，惭愧。一方面是我目前的工作不再关注这些，另一方面是因为跳槽休了个长假。这些天有了一些思考，但是相比之前收获的没有很多。但是也还是腆着脸把所得写下吧。

`find it out`

最近在用springboot重构测试项目依赖管理，但是心里越来越想干票大的。

那什么才算是一个良好的、适配性广、不易腐败的UI自动化测试项目结构呢？

业界已经有一些看法。如我了解到的[Page Object model](https://medium.com/tech-tajawal/page-object-model-pom-design-pattern-f9588630800b). 通过对页面建模，分离功能逻辑与UI点击操作。

我认为， 像一些app， 页面元素、功能都比较简单还好，如果是web page，恨不得把一切东西都塞到同一个页面。这样的话，也需要对组件进行建模。 所以， 在这时需要像前端一样引入component的概念， 对page的功能模块进行细分。

在进行模块的设计时，需要考虑到，在前端中一个模块可能被复用，在ui测试中也是一样的，需要对模块进行复用。

在实操中，我发现不能像开发前端页面一样写ui测试，否则会陷入很大的误区。一是跟随前端页面的建模成本很高，另一方面是经常你只需要关注一点点东西，比如说一个小模块，你并不需要建模出整个页面。
这让我想起一个`香蕉丛林的故事`:

> `当你想要一个香蕉，你会得到拿着香蕉的猴子，最终你会得到整片丛林。`

所以其实很多时候不要太教条，写自动化脚本应当简单。
当你用componentA时， 你不需要通过`new PageA().componentA`来得到，`new ComponentA()`就好了。

可能是我对ui测试的理解不够深吧，总感觉这边是编程设计的`黑洞`,你可以用一些设计来令你的代码优雅，但如果你追求这个，那你只会越来越难过。

ui操作只该被**功能**封装.



[Page Object model](https://medium.com/tech-tajawal/page-object-model-pom-design-pattern-f9588630800b)

在我看这篇文章之前，我没有了解过`Page Object model`, 不过摸索的想法是差不多的。
最终目的是写出这样的代码:(根据代码提示，你会知道什么方法怎么样到了另外一个page)
```java

// 一个确保消息能发成功的脚本
new LoginPage()
    .loginWith("user1", "password")
    .openConversationTo("user2")
    .sendMessage("helloworld")
    .checkMessageSent("helloworld");

interface LoginPage {
    MainPage loginWith(String u, String p)
}

interface MainPage {
    ConversationPage openConversationTo(String u)
}
```


下面是3月3日晚上根据想法随便敲的一点代码，引入状态管理来对被测应用的页面建模。
建议是不用看了， 直接看这里就好: [Page Object model](https://medium.com/tech-tajawal/page-object-model-pom-design-pattern-f9588630800b)


```kotlin
/**
 * Entry point for ui steps
 */
class Workflow {
    companion object {
        val loginService = LoginPageService()

        fun login(user: String): MainState {
            loginService.deepLink(user)
            return MainState()
        }
    }
}

abstract class State

class ConversationState(val conversationName: String) : State() {

    fun sendMessage(message: String) {
    }
}

class MainState : State() {
    val service: MainPageService = MainPageService()

    fun openConversation(name: String): ConversationState {
        service.openConversation(name)
        return ConversationState(name)
    }
}

abstract class Service

class MainPageService : Service() {
    val mainPageUi = MainPage()

    fun openConversation(name: String) {
        mainPageUi.clickItemWithName(name)
    }
}

class LoginPageService: Service() {
    fun deepLink(user: String) {
        // deep link stuff..
    }
}

abstract class Ui

class MainPage : Ui() {
    fun clickItemWithName(name: String) {}
}

class Scrip {
    fun `Login and open conversation successfully`() {
        Workflow.login("user701").openConversation("user702")
    }
}
```