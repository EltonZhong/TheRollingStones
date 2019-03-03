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

## TODO(本篇文章还未完成)
最近在用springboot重构测试项目依赖管理，但是心里越来越想干票大的。

那什么才算是一个良好的、适配性广、不易腐败的UI自动化测试项目结构呢？

`TODO: find it out`

下面是晚上根据想法随便敲的一点代码，引入状态管理来对被测应用的页面建模。


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