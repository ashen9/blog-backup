---
title: Hybrid App技术解析 -- 原理篇
date: 2018-09-12 17:23:24
tags:
---

#现有混合方案
----
Hybrid App，俗称混合应用，即混合了 Native技术 与 Web技术 进行开发的移动应用。现在比较流行的混合方案主要有三种，主要是在UI渲染机制上的不同：

1. 基于 WebView UI 的基础方案，市面上大部分主流 App 都有采用，例如微信JS-SDK，通过 JSBridge 完成 H5 与 Native 的双向通讯，从而赋予H5一定程度的原生能力。
2. 基于 Native UI 的方案，例如 React-Native、Weex。在赋予 H5 原生API能力的基础上，进一步通过 JSBridge 将js解析成的虚拟节点树(Virtual DOM)传递到 Native 并使用原生渲染。
3. 另外还有近期比较流行的小程序方案，也是通过更加定制化的 JSBridge，并使用双 WebView 双线程的模式隔离了JS逻辑与UI渲染，形成了特殊的开发模式，加强了 H5 与 Native 混合程度，提高了页面性能及开发体验。

以上的三种方案，其实同样都是基于 JSBridge 完成的通讯层，第二三种方案，其实可以看做是在方案一的基础上，继续通过不同的新技术进一步提高了应用的混合程度。因此，JSBridge 也是整个混合应用最关键的部分，例如我们在设置微信分享时用到的 JS-SDK，wx对象 便是我们最常见的 JSBridge:

![https://segmentfault.com/img/bVbeQ3l?w=500&h=455](https://segmentfault.com/img/bVbeQ3l?w=500&h=455)

#方案选型
----
任何技术方案的选型，其实都应该基于使用场景和现有条件。基于公司现有情况的几点考虑，在方案一上进一步优化，更加适合我们的需求。

* 需求 Web技术 快速迭代、灵活开发的特点和线上热更新的机制。
* 产品的核心能力是强大的拍照与底层图片处理能力，因此单纯的 H5技术能做的事非常有限，不能满足需求，通过 Hybrid 技术来强化H5，便是一种必需。
* 公司业务上，并没有非常复杂的UI渲染需求，而且 App 中的一系列原生 UI组件 已经非常成熟，因此我们并不强需类似 RN 这样的方案。

因此，如何既能利用 H5 强大的开发和迭代能力，又能赋予 H5 强大的底层能力和用户体验，同时能复用现有的成熟 Native组件，便成为了我们最大的需求点 -- 一套完整又强大的 Hybrid技术架构方案。

#Hybrid技术原理
---
Hybrid App的本质，其实是在原生的 App 中，使用 WebView 作为容器直接承载 Web页面。因此，最核心的点就是 Native端 与 H5端 之间的双向通讯层，其实这里也可以理解为我们需要一套跨语言通讯方案，来完成 Native(Java/Objective-c/...) 与 JavaScript 的通讯。这个方案就是我们所说的 JSBridge，而实现的关键，便是作为容器的 WebView，一切的原理都是基于 WebView 的机制。


![https://mmbiz.qpic.cn/mmbiz_jpg/aVp1YC8UV0fOXBnc9R5yWwuuicR0PfMMwJguS1GMXUFrtpiaVlyuaBlP8HRIMI3bjPt72O2fvOoAx6QwF902PBDQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_jpg/aVp1YC8UV0fOXBnc9R5yWwuuicR0PfMMwJguS1GMXUFrtpiaVlyuaBlP8HRIMI3bjPt72O2fvOoAx6QwF902PBDQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## (一) JavaScript 通知 Native
 
基于 WebView 的机制和开放的 API, 实现这个功能有三种常见的方案：
 
 * API注入，原理其实就是 Native 获取 JavaScript环境上下文，并直接在上面挂载对象或者方法，使 js 可以直接调用，Android 与 IOS 分别拥有对应的挂载方式
 
 * WebView 中的 prompt/console/alert 拦截，通常使用 prompt，因为这个方法在前端中使用频率低，比较不会出现冲突
 
 * WebView URL Scheme 跳转拦截
 
第二三种机制的原理是类似的，都是通过对 WebView 信息冒泡传递的拦截，从而达到通讯的，接下来我们主要从 原理-定制协议-拦截协议-参数传递-回调机制 5个方面详细阐述下第三种方案 -- URL拦截方案。

1. 实现原理

在 WebView 中发出的网络请求，客户端都能进行监听和捕获

2. 协议的定制

我们需要制定一套URL Scheme规则，通常我们的请求会带有对应的协议开头，例如常见的 https://xxx.com 或者 file://1.jpg，代表着不同的含义。我们这里可以将协议类型的请求定制为:

xxcommand://xxxx?param1=1&param2=2

这里有几个需要注意点的是:

(1) xxcommand:// 只是一种规则，可以根据业务进行制定，使其具有含义，例如我们定义 xxcommand:// 为公司所有App系通用，为通用工具协议：

xxcommand://getProxy?h=1
而定义 xxapp:// 为每个App单独的业务协议。

xxapp://openCamera?h=2

不同的协议头代表着不同的含义，这样便能清楚知道每个协议的适用范围。

(2) 这里不要使用 location.href 发送，因为其自身机制有个问题是同时并发多次请求会被合并成为一次，导致协议被忽略，而并发协议其实是非常常见的功能。我们会使用创建 iframe 发送请求的方式。

(3) 通常考虑到安全性，需要在客户端中设置域名白名单或者限制，避免公司内部业务协议被第三方直接调用。



 