title: Web Push 如何工作
date: 2017-04-28
tags: 
 - frontend
 - web push
 - Push API
 - pwa
comments: true
categories:
 - frontend
 - pwa
---


在了解 API 之前，让我们整体了解一下 Web Push 的工作流程，总共分为三步

1. 编写浏览器端订阅消息的逻辑
2. 后端服务调用 API 触发推送消息到客户端
3. 当消息推送到达客户端时，服务工作线程接受 `Push Event` ，然后调用 `Nofitication` 展示通知

下面我们深入剖析一下这三个步骤。

## 第一步，编写订阅逻辑

第一步是订阅消息。

订阅一个用户需要先完成两件事情，第一是，获取推送消息的权限，第二，从浏览器获取 `PushScbscription` 对象。

`PushSubscription` 对象包含我们需要用来推送消息的所有信息，也可以简单的认为它是用户设备的 ID。

[Push API](https://developer.mozilla.org/zh-CN/docs/Web/API/Push_API) 已经封装好了内部逻辑。

在订阅之前，我们需要先生成一组 `application server keys`，`application server keys` 也叫 `VAPID keys`，是唯一标识我们的服务的 keys，推送服务通过这组 keys 判断是哪个服务推送消息给用户。

在用户确认订阅，并且获取到 `PushSubscription` 对象之后，我们需要把 `PushSubscription` 的详细信息发送到我们的服务器，服务器会保存这些信息，并且用它来发送消息给用户。

 总结为三步：

1. 获取推送消息的权限
2. 获取 `PushSubscription` 信息
3. 发送 `PushSubscription` 信息到服务器

![Web Push 的三个步骤](https://boscdn.baidu.com/v1/assets/browser-to-server.svg)

## 第二步，发送消息

当我们想给用户推送消息的时候，我们需要调用`推送服务`的 API，需要将发送的数据、发送给谁和怎么发送的字段都发给推送服务，一般情况下，这个步骤是由我们的服务器来完成。

有些问题你可能想问：

1. 什么是推送服务？
2. 这个 API 是什么？是 JSON，还是XML，还是其他什么？
3. 这个 API 干了什么？

### 什么是推送服务

推送服务接受网络请求，并且校验，然后将消息推送给目标浏览器，如果当前浏览器处于离线状态，消息会被存在队列里，直到目标浏览器上线。

每个浏览器可以选择他们想要的推送服务，这是 Web 开发者无法控制的，这并不是问题，因为每个推送服务都实现了同样的 API，这意味着你不需要关心浏览器使用哪个推送服务，你只需要确保你的 API 调用是正确的。

`PushSubscription` 中的 `endpoint` 是推送服务的地址，你可能需要关心这个值，下面是`PushSubscription` 的一个例子，包含所有的值。

```javascript
{
    "endpoint": "https://random-push-service.com/some-kind-of-unique-id-1234/v2/",
    "keys": {
        "p256dh":
"BNcRdreALRFXTkOOUHK1EtK2wtaz5Ry4YfYCA_0QTpQtUbVlUls0VJXg7A8u-Ts1XbjhazAkj7I99e8QcYP7DkM=",
        "auth": "tBHItJI5svbpez7KI4CCXg=="
    }
}
```

这个例子中，**endpoint** 的值是 *https://random-push-service.com/some-kind-of-unique-id-1234/v2/*，推送服务的地址是 `random-push-service`，`some-kind-of-unique-id-1234` 唯一表示每个 `endpoint`，也就是说 `endpoint` 对每个用户都是唯一的。

**keys** 后面会讲到。

### 这个 API 是什么？

前面讲到，每个推送服务都实现相同的 API，那个 API 就是 [Web Push Protocal](https://tools.ietf.org/html/draft-ietf-webpush-protocol)，它是 IETF 的标准，规范对推送服务的 API 调用。

API 规范要求设置特定的请求头。

### 这个 API 干了什么？

API 提供了发送消息给用户的方法，并且告诉了我们如何发送。

发送给用户的消息必须是加密后的，这可以防止中间人拦截并获得消息，浏览器的选择在这里的地位显得很重要，因为它完全可以给不安全的推送服务开后门。

当服务器发送了一条消息给推送服务，推送服务将消息放在队列中等待直到用户的浏览器连接服务，我们可以给推送服务指定一些指令告诉它如何处理在队列中等待的消息。

这些指令包括以下几种：

* 消息的存活时间 - 定义消息在队列中等待的最长时间，超过会被删除或者不发送给用户
* 定义消息的紧急程度 - 在用户设备电池续航能力不够的情况下，只投递重要的消息
* 给消息定义一个 `topic`，新消息会覆盖当前正在队列中的同 topic 的消息

![服务器发送消息的流程](https://boscdn.baidu.com/v1/assets/server-to-push-service.svg)


## 第三步，推送消息到用户的设备
 
 当我们发送一个消息到推送服务，推送服务会一直保存消息在服务器上，这些消息只有两种命运，到达或者销毁
1. 用户的设备连接推送服务，会将消息推送给用户
2. 消息过期，推送服务会将这条消息删除，不会投递

当浏览器接收到消息，解密之后，会在你的 `service worker` 中抛出一个 `push` 事件。

[service worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) 是一个特殊的 JavaScript 文件，浏览器可以在你没有打开当前页面的情况下执行这个文件，甚至可以在你关闭浏览器之后继续执行，`service worker` 也有一些可用的 API，这些 API 在 `service worker` scope 外不能被调用。

`service worker` 在接收到 `push` 事件之后可以选择在后台执行一些任务，可以发送统计分析请求、缓存页面，也可以显示消息给用户。

![接收消息后浏览器的处理流程简图](https://boscdn.baidu.com/v1/assets/pwa/push-service-to-sw-event.svg)

这就是推送消息的整个流程。


[Google 的原文](https://developers.google.com/web/fundamentals/engage-and-retain/push-notifications/how-push-works)
