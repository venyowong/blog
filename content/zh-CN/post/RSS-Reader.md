---
title: RSS 阅读器的推荐以及 Orleans 框架的介绍
date: 2020-02-05
tags:
  - .net core
  - rss
  - orleans
---

# 写在前面：
- 本文推荐的 RSS 阅读器为 [Inoreader](https://www.inoreader.com/dashboard)
- [Orleans](https://github.com/dotnet/orleans) is a cross-platform framework for building robust, scalable distributed applications

## RSS 阅读器的推荐

简易信息聚合（也叫聚合内容）是一种RSS基于XML标准，在互联网上被广泛采用的内容包装和投递协议。RSS(Really Simple Syndication)是一种描述和同步网站内容的格式，是使用最广泛的XML应用。RSS搭建了信息迅速传播的一个技术平台，使得每个人都成为潜在的信息提供者。以上简介来自[百度百科](https://baike.baidu.com/item/rss/24470)。

在我开始习惯于订阅 RSS 之前，在空余时间想要获取一些技术文章时，需要打开各种社区(博客园、开源中国、CSDN等)去寻找自己想要了解学习的技术，而且还需要隔一段时间就到各个社区首页检查是否有新的文章。随着时间长了，就开始变得对频繁在不同社区来回切换浏览感到麻烦了，想着是否有一个应用可以聚合这些分散的信息，便于浏览。忘记是本来就对 RSS 有点印象，还是在搜索解决方案的时候才发现了 RSS 这样的技术了。反正自从我开始订阅 RSS 以后，就没怎么主动去社区里浏览信息，寻找技术文章了。

虽然 RSS 在我看来是一个很高效、实用的工具，但是当我在搜索 RSS 客户端的时候，发现并没有什么特别好的选择。最开始我使用的一款在线 RSS 阅读器叫做[一览](www.yilan.io)，链接是我凭印象写的，点开链接可以看到现在已经找不到服务了。RSS 竟然是一个如此冷门的技术，当一览站点关闭以后，我真的如此感叹过。后来有一段时间没怎么通过 RSS 获取信息了，因为找不到符合我习惯的客户端，后来某一天，我实在觉得自己空闲时间有点多，需要利用起来，于是下定决心自己做一个[客户端](https://github.com/venyowong/RssReader)。点开左边这个链接之后会发现项目已经归档了，因为我今天用自己写的这个客户端浏览文章时，在少数派文章中发现了一个很好用的阅读器[Inoreader](https://www.inoreader.com/dashboard)。

RSS 阅读器推荐就说这么多，下面介绍一下 [Orleans](https://github.com/dotnet/orleans)，因为我本来打算用 Orleans 框架实现我自己写的 RSS 阅读器的 Web 版的用户服务，但是现在已经不需要了。

## Orleans 框架

[Orleans](https://github.com/dotnet/orleans) 是由 [Microsoft Research](http://research.microsoft.com/projects/orleans/) 创建的一个 .Net 框架。基于该框架能够快速创建具有鲁棒性、可扩展性的分布式应用。Orleans 使用了一种叫 Virtual Actors 的编程模式，它可以为程序员免去多线程、程序容错、资源管理等复杂性，可以让程序员更加地专注于实现业务逻辑，更好地实现应用服务。得益于 Virtual Actors，基于 Orleans 实现的服务能够很容易地做到代码上的简洁，具体示例代码可以参考[官方文档](http://dotnet.github.io/orleans/Documentation/index.html)，或者我自己写的[示例](https://github.com/venyowong/orleans-silo)。

### 分布式

[Orleans](https://github.com/dotnet/orleans) 的分布式特性依赖于服务注册、发现，说到服务注册，往往都会联想到各种注册中心(Consul、etcd、zk)，但 Orleans 官方使用的注册中心均为数据库(SQL Server、MySQL、PostgreSQL、Oracle)，当然只要实现了 Orleans 的接口，其他的基础设施也都能作为 Orleans 的注册中心。我使用 MySQL 作为注册中心，从体验上来说，服务的搭建过程是非常流畅的。由于我使用了随机数作为服务端口，因此我只需要配置一个数据库连接字符串，便可以在一台或多台机器，启动多个服务实例，服务之间的通信问题 Orleans 能完美解决，再也不用在配置每个服务的地址了。

### Grain

[Grain](http://dotnet.github.io/orleans/Documentation/grains/index.html) 是 Orleans 中核心单元，相当于一个服务对象，程序员只需要创建一个 Grain 继承某个服务契约(接口)，实现每个方法的业务逻辑，就可以实现一个具有鲁棒性、可扩展性的服务了。在客户端获取 Grain 的时候，需要传 id 参数，也就是说 Grain 能通过 id 将请求隔离开，而且 Orleans 能保证同一个 Grain 实例顺序处理请求，因此在写业务逻辑时，可以不需要考虑多线程所带来的复杂性。而且每个 Grain 实例都可以有一个 State 对象，该对象可通过 Orleans 进行[持久化](http://dotnet.github.io/orleans/Documentation/grains/grain_persistence/index.html)，因此 Grain 也可以很简单地实现有状态服务。

### Stateless Worker Grain

[Stateless Worker Grain](http://dotnet.github.io/orleans/Documentation/grains/stateless_worker_grains.html) 与 Grain 的差别在于，前者在同一个服务实例中可以有多个对象。接受到请求的 Silo(Grain 服务的宿主服务实例) 会在本地创建 Stateless Worker Grain 实例，当实例处于 Busy，则会创建新实例，直到上限，上限默认为内核数量。虽然名字叫做 Stateless Worker Grain，但是它也可以有 State 对象，只是在使用 State 对象的时候，需要考虑并发读写问题，因为 Stateless Worker Grain 可以有多个实例，而且它们有同一个 id，因此它们所持有的 State 对象是同一个数据源。

### Reentrant

当 Grain 中的方法被标注上 Reentrant 特性时，该方法将会变成可重入。经过我个人实验，发现当使用 Thread.Sleep 阻塞线程时，该特性无法生效。当使用 await 时，请求可重入同一个 Grain 实例，即在一个请求未完成前，可继续处理下一个请求，此时，`Grain activations are single-threaded` 特性将会失效，需要在代码中自行处理并发。

对于 Orleans 的介绍就这么多，因为我并没有将所有特性的文档都看完，Orleans 还有许多其他特性，感兴趣的朋友可以自行前往查看[官方文档](http://dotnet.github.io/orleans/Documentation/index.html)。在此，我自己总结了几个自认为还算比较合适的要点：
- 优先考虑使用 Stateless Grain，利用 Orleans 可伸缩的优势提高接口吞吐量和效率
- 使用 Stateless Worker Grain 时，尽量实现无状态服务，即尽量不使用 State 对象
- 在 Stateless Worker Grain 中使用 State 对象时，应是读多写少的服务，并且利用分布式读写锁保证状态的实时性，或是定时更新状态
- 频繁写入 state 的服务，应该使用普通的 Grain 以保证状态的实时性以及逻辑的简洁