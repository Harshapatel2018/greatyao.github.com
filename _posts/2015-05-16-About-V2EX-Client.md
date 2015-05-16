---
layout: post
title: V2EX客户端
tagline: null
keywords: V2EX
categories: [技术]
tags: [开源, Android, JAVA]
group: archive
icon: leaf
---

{% include codepiano/setup %}

## 前言
最先接触到[v2ex](http://www.v2ex.com)是三四年前，作为一个技术人员，很快就被其深深吸引住了。不过大部分时间只是潜水而已，基本没怎么发言。

Android和iOS市场上的V2EX客户端我基本上都下载安装过，有一段时间，V2EX被墙，大多或多或少的受了牵连，不过也有维护者随之作出了修改。其中给我印象最深的是[iOS的V2EX](https://itunes.apple.com/us/app/v2ex-chuang-yi-gong-zuo-zhe/id898181535?ls=1&mt=8)和Android的[每日V2EX](https://play.google.com/store/apps/details?id=com.yugy.v2ex.daily)。在这里要谢谢这两位开发者，我有幸从他们的开源代码中借鉴了很多。

究竟是什么缘由促使我自主开发Android客户端，已经很模糊了。我当时在Github上给一些V2EX客户端Pull Request后，总觉得有些设计可以更优雅一点，于是就自己动手写了。对于Android开发经验基本为零的我，挑战还是不小的。所以我上文特别提到到了要感谢开源社区，不至于让我无所适从。前前后后花了两个月左右的时间，V2EX Android客户端终于出炉了。

## 功能
目前已经完成的功能包括：
 
 - 最新、最热话题查看
 - 所有节点查看和按照拼音首字母排序查找
 - 登录
 - 用户资料查看
 - 话题创建和回复
 - 未读提醒查看
 - 话题和节点收藏
 - 支持节点、话题、用户三个Scheme

绝大部分数据是通过调用V2EX的Json
API解析得到的，少部分涉及到用户个人信息则是通过Http模拟浏览器请求页面。但是Json API调用有严格的调用次数和时间限制，为了避免这个问题，我将数据缓存到文件系统中，如果用户不手动刷新，则会直接从缓存中读取的，当然这并不能解决根本问题，每个小时180次请求的警戒线还是很容易就突破，这时候服务器返回403禁止访问，会很大的影响用户体验。我在后续版本中会考虑绕过Json API用模拟浏览器访问来代替。

## 开发相关
开发用的是 Android Studio，除了编译速度感觉略慢一点点，就没有什么其它的大问题了，而且智能提示更智能，此外用 gradle 做库依赖确实方便，真的可以甩 eclipse 几条大街了。

V2EX客户端里面的列表用到了Android兼容库

    compile 'com.android.support:cardview-v7:21.0.3'
    compile 'com.android.support:recyclerview-v7:21.0.3'
刚开始对于V2EX的主题列表和回帖列表，我都是用ListView实现的。不过后来发现总存在一些问题，在Nexus 5上没什么问题，但是在其他手机上列表会出现一些锯齿。于是我用RecyclerView来重新实现了一遍。CardView则对每个话题Item进行卡片式布局。
<img src="https://raw.github.com/greatyao/v2ex-android/master/snapshots/hot.png" width="400" height="700"/>


这是V2EX-Android中用到的第三方库：

    compile 'com.nostra13.universalimageloader:universal-image-loader:1.9.3'
    compile 'com.astuetz:pagerslidingtabstrip:1.0.1'
    compile 'com.loopj.android:android-async-http:1.4.6'
    compile 'com.github.mrengineer13:snackbar:1.1.0'
    compile 'com.melnykov:floatingactionbutton:1.3.0'

- android-async-http

    封装了 http 请求，直接支持 json，gzip 压缩，相当省事。

- universal-image-loader

    异步图像加载，缓存和显示，如果你想要在界面上显示网络图片，那么赶紧使用它吧。

- pagerslidingtabstrip

    交互式页面指示器控件，完美配合ViewPager控件。

<img src="https://raw.github.com/greatyao/v2ex-android/master/snapshots/favor.png" width="400" height="700"/>

- snackbar
     Snackbar 是 Material Design 下的一个组件，这是模仿Snackbar的效果实现了一款兼容5.0系统以下的Snackbar。
    

- floatingactionbutton

    浮动Action Button控件，完美配合ListView、RecyleView。


除此以外，还使用了

- [友盟](http://www.umeng.com)的SDK作统计分析和自动更新
- [BadgeView](https://github.com/stefanjauker/BadgeView)作数字提醒
- [Pinyin4J](http://pinyin4j.sourceforge.net) 将汉字转化为对应的拼音字母
