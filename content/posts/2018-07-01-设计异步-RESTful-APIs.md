---
date: "2018-07-01 19:34:30"
toc: true
id: 253
slug: /posts/designing-asynchronous-restful-apis
aliases:
    - /archives/2018/07/253/
tags:
    - Web
    - 异步
    - RESTful
    - 设计
    - 最佳实践
title: Designing Asynchronous RESTful APIs
---

RESTful API 已经不是什么新奇玩意儿了。HTTP 的无状态性让我们在处理交互式 CS 应用中不得不使用诸如 Session、Cookie 等技术。那么我们的问题产生于：如果客户端发出的请求的在服务端需要较长的时间处理，我们应该如何设计我们的 RESTful API？这次就来聊一聊异步 RESTful API 的设计。

<!--more-->

## 异步 RESTful 的场景

一个普通的 Web 应用通常不会遇到需要异步 RESTful API 的场景，它们的交互逻辑固定为：

- 客户端请求 --> 服务端程序与数据库交互 --> 响应客户端。

现在假设我们的业务非常的复杂，需要后端进行长时间的处理（比如，五分钟），那么我们应该如何设计 RESTful API 来与客户端进行异步任务交互呢？

我们想象一个异步的事务过程：

1. 客户端请求 -> 服务端接收到请求 -> 异步启动任务 -> 响应客户端任务已接受
2. 客户端请求 -> 服务端收到任务进度查询请求 -> 查看任务进度 -> 响应客户端任务的执行状态

## 异步 RESTful 产生问题

- 接口幂等性的保证是让客户端生成还是服务端分配？
- 为什么不选择 WebSocket？
- 服务端的异步任务如何对任务进度进行记录？
- 如何让客户端从请求结果中识别异步任务成功还是失败？
- 如果异步任务失败了，服务端需要对异步任务涉及的资源作何处理？
- ...

## 接口设计

当上面这些问题思考清楚后，我们就能成功的设计出异步 RESTful API 了。先说结论：

- GET `/api/v1/task`  返回一个服务端分配的唯一标识
- POST `/api/v1/task/:uuid` 当客户端获得唯一标识后，可以通过请求此接口来提交异步任务，并返回 202 Accepted
- GET `/api/v1/task/:uuid` 当客户端提交异步任务后，可以通过此接口来查询异步任务的执行结果，未完成时返回进度
- PUT `/api/v1/task/:uuid` 当客户端提交异步任务后，可以通过此接口来更新一些异步任务的信息
- DELETE `/api/v1/task/:uuid` 在有需要的场景下，可以通过此接口来让客户端主动发起销毁异步任务（包括异步任务产生的资源等数据），当销毁任务执行后 `/api/v1/task/:uuid` 所有的操作均不能使用，返回无效事务的错误。

首先我们来看第一个问题：**接口幂等性的保证让客户端生成而是服务端分配？**

通常我们会看到一些接口设计的幂等性支持客户端在 POST 请求 Body 中包含一个保证幂等性的唯一标识。这是一种做法。另一种做法是，通过 **先通过一个接口让服务端分配一个异步任务的事务 ID，然后客户端收到此 ID 后凭借此 ID 进行后续操作。这种做法相较于前一种做法的好处在于：服务端完整的掌控的所有的异步事务。** 思考这样一个场景：客户端 A 和 客户端 B 同时使用了相同的标识符来做幂等性的保证，那么服务端必须鉴别这两个相同的标识是来自不同的客户端实体。这可以通过在请求中增加额外的身份验证来完成，但这进一步增加了服务端的复杂度以及客户端与服务端之间的交互成本。

**为什么不选择 WebSocket？**这个问题很好回答，我们来思考一些比较极端的情况。首先明确我们的异步事务场景是：**客户端提交一个异步任务，在未来的某个时间段，客户端需要知道这个异步任务的执行状态。**那么 WebSocket 会发生什么问题？假设我们同时接受了百万级别的异步任务，那么服务端必须维护百万级别的长连接从而通知客户端异步任务的执行状态。试想：一百万的长连接和一百万的并发，哪种方式对服务器产生的压力相对较小？使用 WebSocket 后还会引入很多额外的诸如断线重连、主动关闭、主动关闭失败等等错误处理，**一方面增加了客户端和服务端之间的交互的开发成本，另一方面还引入了额外的系统复杂性。**

好，那么我们现在可以任务客户端已经成功完成了异步任务的提交。那么客户端如何得知所提交异步任务的执行进度，又应该以什么样的策略来主动对服务端发送查询请求，以及如何鉴别一个异步任务执行失败与否？进度如何？

第一个问题：**如何得知提交异步任务的执行进度。**这个问题看似是一个比较难解的问题，因为服务端的任务调度器也不知道这个任务要多久才会执行完毕。事实上这样的场景只会发生在某个任务的第一次执行中，服务端可以对不同的任务进行鉴别，自然也可以对不同任务的执行时间进行统计。例如：客户端发起请求需要服务端执行任务 A ，服务端查询统计记录了解到任务 A 在所有历史记录中平均需要 158 秒，那么服务端可以以此为标准，直接假设任务 A 的执行时间为 158 秒，那么当客户端的查询请求到达后，比较请求提交时间和查询时间，从而推算任务进度，具体计算公式为：**任务进度 = （查询请求到达时间 - 创建任务时间）/ 平均任务时间。如果任务进度计算结果大于等于 1，则任务进度为 99%。当异步任务执行完毕后，再将任务进度主动更新为 100%。**

第二个问题：**客户端以什么样的策略来主动对服务端发送查询请求。**一个比较「弱智」的做法就是定时轮询。这个做法不应该被采纳的原因是，定时轮询需要消耗大量的客户端发送请求资源和服务端处理资源。我们仍然可以通过第一个问题中提到的统计模块来让服务端主动告知客户端，什么时候再发送查询请求。在 HTTP 请求 Header 中有一个 [Retry-After](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After) 字段。那么，**当一个客户端任务请求到达后，服务端首先检查任务是否有历史记录，如果有则统计平均任务完成时间（或者最长完成时间），并在 `Retry-After` 中附上此时间，从而客户端可以得知在何时再发送历史请求，而无需进行不必要的定时轮询浪费资源。如果没有，则设定一个默认的轮询时间即可。**

第三个问题：**客户端如何得知一个异步任务执行失败。**在我们这套接口中，如果一个异步任务失败，服务端会被动执行销毁程序，那么对应的 uuid 接口会被禁止。这样客户端看似永远无法知道任务的执行状态，因为对它而言这时是处于查询任务执行状态的阶段。这又两个办法，**一个是客户端在查询请求中始终遵循服务端 `Retry-After` 的原则，如果某一次 `Retry-After` 后请求失败，则可以认为任务是执行失败的**；另一个方法是，客户端记录每次 `Retry-After` 查询到的任务进度，如果接口在进度小于 100% 的情况下失败了，那么也可以认为任务是失败的。

最后，**如果异步任务失败了，服务端需要作何处理？**这个问题相对上面的几个问题就比较简单了。答案是服务端需要有被动销毁任务。当一个异步任务失败后，我们**需要设计一个被动的销毁任务，让异步任务中涉及的相关操作进行回滚或者删除**，例如当客户端在申请某个创建资源后，却没有创建成功，则销毁任务应将此资源创建过程中涉及的中间产物一并销毁。

## 总结

好了，解决了这些问题我们基本上完成了一整套异步 RESTful API 的设计工作。我们来总结一下异步 API 的几个基本要点：

1. 服务端分配异步事务的唯一标识

2. 服务端的对客户端请求的异步事务执行进行统计

3. 服务端需要实现异步事务失败的被动被动销毁任务

4. 服务端需要在返回请求时一并告知客户端下一次主动与服务端交互的时间

   