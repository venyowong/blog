---
title: .net 经验总结
date: 2021-12-30 17:22:38
tags:
  - .net
  - c#
---

先做个自我介绍，我是 2015 年大四实习开始学习的 .net，如果从 2016 年毕业开始算，到目前为止已有 5 年的 web 开发相关经验了，自认为并不是技术很厉害的人，但是也有一点个人的经验，希望能够帮助到其他人。

# 能脱坑就尽快吧

对于 web 开发，我想要简单对比下 c#、java、go 这三门语言，主要是个人稍微比较了解并不代表其他语言不适合于做 web 开发。三门语言的各方面对比相信网上都有很多相关的文章，在此不再赘述，主要想表达一些比较主观的个人观点。

go 语言个人认为是一门比较现代化的语言，无论是从语法上还是从设计上都感觉比其他语言来得简洁。
- 错误处理：conent,err:=ioutil.ReadFile("filepath")

    c#、java 的错误处理方式都是需要通过 try catch 来捕获，但是 go 只需要在有可能报错的方法后增加一个 err 变量。虽然 c# 可以使用在方法内部 try catch，然后使用 tuple 封装错误的方式来实现类似的效果，但使用起来始终没有 go 这么优雅。
    ```
    public static (string, Exception) ToJson(this object obj)
    {
        if (obj == null)
        {
            return (string.Empty, new ArgumentNullException(nameof(obj)));
        }

        try
        {
            return (JsonConvert.SerializeObject(obj), null);
        }
        catch (Exception e)
        {
            return (string.Empty, e);
        }
    }

    var (json, e) = obj.ToJson();
    ```

    **顺带一提：** V 语言的错误处理是我认为目前为止最简洁的方案，但目前这门语言仍在开发中。
    ```
    fn (r Repo) find_user_by_id(id int) ?User {
      for user in r.users {
        if user.id == id {
          // V automatically wraps this into an option type
          return user
        }
      }
      return error('User $id not found')
    }

    user := repo.find_user_by_id(10) or { // Option types must be handled by `or` blocks
      return
    }
    ```
- 启动多线程(协程)的方式

    go 启动协程的方式特别简洁且优雅 `go function()`，虽然 c# 也可以很快创建线程 `Task.Run(Function)`，但 go 的这种设计还是更加让我喜欢。
- channel 的设计

    channel 是一种用于在协程之间相互通信的技术，这让我想到了大学开发安卓时 UI 线程与子线程的通信，显而易见，channel 更加的简洁、优雅。

而 c# 与 java 之间，大部分人都会觉得这两门语言比较相似，优缺点大概就是 C# 语法糖比较多开发比较快，java 社区庞大类库多开发也很快，优点在此不多赘述，个人认为 .net 类库没有 java 丰富，目前 java 的 web 开发基本离不开 spring，而 spring 个人认为有点过度设计。

除了以上说到的一点浅显的对比，还有很关键的一点就是就业市场，以下是我在拉勾网上的搜索结果。
![java 岗位](https://s4.ax1x.com/2021/12/30/TW19Nd.png)
![.net 岗位](https://s4.ax1x.com/2021/12/30/TW1ZDS.png)
![go 岗位](https://s4.ax1x.com/2021/12/30/TW1Dv6.png)

很明显，.net 如今的就业市场是无法与 java 相比的，而 go 作为后来者在上海也已经快赶超 .net 了，基于以上种种，**个人建议对于已经使用 C# 作为工作语言多年的人，可以继续深入学习，尽管就业市场并不乐观。对于刚入坑不久的人，建议可以了解一下其他语言，尽快脱坑。**

# 一点小建议

对于往后还将从事 .net 开发的人，我有一点小小的建议，希望能对你有所帮助。

## 尽可能少用第三方框架

个人建议在开发中尽可能少去使用各种第三方框架，并不是说第三方框架不够优秀。
- 官方框架已足够优秀

    这边说的是 asp.net core，对于 .net framework 以及 MVC 不再做评价，因为这些技术已不再是主流。从 asp.net core 1.0 开始，.net 开发团队就热衷于使用 owin 的启动方式，类似于 `app.Start<Startup>();` 然后在 Startup 中去做各种各样的配置，这点只是个人观察所得，并没有看到官方有此类描述。这种服务启动方式，比过去的 asp.net 项目更加直观清晰，反正让我感觉很舒适。此外，asp.net core 的依赖注入也是个特别好的设计，对象统一在 Startup.ConfigServices 注册，并且只通过构造函数注入，这可以很好地规范项目代码。
- 第三方框架可能说没就没
  
    15 年刚开始学习 .net 的时候，发现了一个三方框架叫 [Nancy](https://github.com/NancyFx/Nancy)，当时感觉这个框架比官方的 MVC 框架优雅多了，然而万万没想到的是，这么多星的一个仓库现在已经不再维护了。可能有些人会觉得反正只要不出 bug、项目足够稳定就可以，但是项目不再维护意味着框架不再会有新功能，将来某个时刻发现框架已难以满足开发需求，那时候就只能重构迁移至其他框架了。

    我现在维护的一个老项目就使用了一个很老的第三方开发框架，现在去搜索已经找不到代码库了，博客园上有一点相关的文章，giee 上似乎有这个框架的私有库，也不知道当时这个框架是怎么弄到手的，反正现在就只有几个 dll，出 bug 了要看源码只能反编译，而恰好还真被我发现了一些 bug。对于这种不开源、维护状态未知的框架，都有人敢使用，我是佩服他的勇气的。强烈建议，千万不要在工作中随便使用什么乱七八糟的第三方框架。

## 自建项目模板

也许有人会说用第三方框架能快速开发，项目能够更快地上线，但是个人认为那种所谓"开箱即用"的开发框架都属于过度设计，将许多东西都揉在一个项目里面，然后还在框架中加入了很多具有个人特色的开发习惯，比如DDD。若为了能达到快速开发，其实每个人都通过为自己[建立一个项目模板](https://docs.microsoft.com/zh-cn/dotnet/core/tutorials/cli-templates-create-project-template)而轻松达到这个目的。

将自己常用的一些组件以及对组件的适当封装都写到项目模板中，在下次需要新建项目的时候，直接基于此项目模板就可以快速继承自己积年累月总结出来的一套最适合自己的开发模式。我个人就在 github 上维护了一个自己的[项目模板](https://github.com/venyowong/ProjectTemplate)，如果你觉得这种方式可行的话，可以参考官方文档以及我的模板去搭建一个自己的项目模板。

# 最后

以上所有内容都是比较主观的观点，若有不对之处，还请指出，望多多包涵。邮箱：venyowong@163.com