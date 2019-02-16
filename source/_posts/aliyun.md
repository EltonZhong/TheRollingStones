---
title: '绕开阿里云域名备案: 升级到https小结'
desc: '如何绕开阿里云&腾讯云烦人的备案?'
cover: imgs/theworst.jpg
tags:
  - Code
categories:
  - Code
date: 2019-02-12 20:02:21
---


## 起因：
博客之前一直serve在github.io上面，由于github访问速度实在是慢，所以打算迁移到自己买的阿里云服务器上。

但是，当我把自己的域名解析到阿里云服务器上时， wtf，返回的页面居然是一个阿里云的页面，要求我对服务器进行备案。

备案？emmm...可以接受。我点进去， 结果发现下面这些东西。。。。
![image.png](https://user-gold-cdn.xitu.io/2019/2/12/168e1bb89aa868a9?w=1240&h=670&f=png&s=250303)

`20个工作日！还能不能让人好好玩耍了！`

所以如何绕开烦人的备案？
我发现当我只是通过ip访问时， 一切正常； 当我使用域名访问时，则返回阿里云的备案页面。而我的域名又是在腾讯云买的， 所以可以断定：`这是一起http劫持事件`。

阿里云劫持了我的http请求，判断是通过域名访问， 则篡改我的http响应。如何解决呢？https可以完美解决这种问题。

> 这里大家好像都奇怪为什么https可以绕过备案，我补充一下：https会对数据进行加密，可以避免中间商对数据进行修改。经常发现登陆某些网站被植入联通的广告，实际就是运营商修改了响应。如果换成https，运营商将无法篡改你的响应。


## Let's Encrypt
黑喂狗，下一步就是要让服务可以通过https来访问。

那么问题来了，https要求有一张受浏览器信任的证书。这时， [Let's Encrypt](https://letsencrypt.org/) 作为一个免费、受信任广的证书签发机构，自然成为了我的首选。

但是，就在这时，我踩了两个坑。

### 腾讯云域名解析
![image.png](https://user-gold-cdn.xitu.io/2019/2/12/168e1bb89abe170f?w=982&h=882&f=png&s=106810)

let's encrypt是一家境外的CA， 所以在选择线路类型时需要选择默认。我当时选择了境内而不自知，费了一番功夫才发现原来境外解析不了这个域名。

### 阿里云的篡改http响应
let's encrypt 给你签发证书的条件是证明这个域名是你的。
有两种方式， 一种是webroot， 即你在你的域名解析到的服务器的80端口上serve一个let's encrypt 指定的页面。即我访问http://www.example.com/letsencrypt，返回的需要是let's encrypt指定的文本。
我一开始就是用的这种方式，使用了let's encrypt 的certbot的webroot模式去验证。结果发现`TM返回的response被阿里云劫持了`。发现这里绕了个圈，又回来了。

还好let's encrypt提供了另一种方式：域名解析指定的txt。这就好办多了， 类似下面这样配置域名解析就可以了：![image.png](https://user-gold-cdn.xitu.io/2019/2/12/168e1bb89ad6f943?w=1240&h=118&f=png&s=6110)

然后你可以获得let's encrypt给你签发的证书， 在`/etc/letsencrypt/live/`域名/路径下。



## 上nginx配置
最后用nginx 为你的服务serve一下即可。上nginx配置：
```nginx
server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;
        root         /root/workspace/eltonzhong.github.io;

        ssl_certificate "/etc/letsencrypt/live/therollingstones.cn/cert.pem";
        ssl_certificate_key "/etc/letsencrypt/live/therollingstones.cn/privkey.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
}
```

## The end
总体还是蛮简单的，只要不像我一样愚蠢地踩到腾讯云域名解析路线的坑即可。
谢谢阅读，如果觉得有帮助，可以点个赞吗~谢谢！
本篇文章在掘金上也拥有一定的访问量，[详情请见掘金](https://juejin.im/post/5c62bf5f51882562c5505fa4) 。