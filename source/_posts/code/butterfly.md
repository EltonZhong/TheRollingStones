---
title: 用手机运维🙊
desc: '今天又起晚了，昨晚还没带电脑回去。结果服务挂了，被一顿猛锤。。
用ngrok + butterfly搞了个手机terminal，不好意思，以后我就用手机运维了。(博君一笑)'
cover: imgs/theworst.jpg
tags:
  - Code
categories:
  - Code
date: 2019-01-07 21:44:10
---

# 用手机运维

先上效果图：
![image.png](https://upload-images.jianshu.io/upload_images/14858485-8862c8943df03bfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



方案是 [内网穿透工具ngrok](https://dashboard.ngrok.com/get-started) + [web终端butterfly](https://github.com/paradoxxxzero/butterfly) 

整体思路是先通过butterfly在你电脑上打开一个web终端，
通过内网穿透工具ngrok把你在本地电脑上的服务代理到ngrok的服务上。 小孩没娘，说来话长，我们一步步来。

## Butterfly

依次把下面几个命令跑完就成，这一步比较简单：

```sh
pip install butterfly
butterfly.server.py --ost=localhost --port=57575 --unsecure
```
记住端口号哦~ 

`在浏览器打开 http://localhost:57575`

你就可以开始玩web 命令行了

## Ngrok

当然web命令行不能满足你
所以你需要这个 ngrok这个神器

ngrok 稍微麻烦点，你需要去注册个账号，但是总体流程还是比较简单。 注意这个还是个闭源收费项目，免费也够用了就是
跟着教程来，不多赘述，比较简单。

`最后跑   ngrok http 57575`
然后他会给一个公网地址给你，类似 https://e820338d.ngrok.io这种。 ngrok不仅能代理http请求，websocket请求也能代理，并且可以展示http请求内容，可以说是非常优秀了，不愧是收费软件。

然后你通过手机访问这个地址，就可以获得一个终端了。 `密码是你的当前用户的密码。`

不过可能速度稍微吃力，速度快点可能收费吧，或者他的服务器不在中国。 不过没关系，差不多了。

就是这么简单！