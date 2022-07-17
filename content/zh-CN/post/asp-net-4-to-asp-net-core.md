---
title: ASP.NET 4 迁移到 ASP.NET Core 的部分改变
date: 2020-01-28
tags:
  - .net core
  - asp.net
  - asp.net core
---

[官方迁移链接](https://docs.asp.net/en/latest/migration/index.html)

接下来是正文(会涉及到 .Net Core 的迁移)：
- 从 Type 中获取 Attribute 特性
    原来是这样：
    ```
    type.GetCustomAttributes()
    ```
    现在是这样：
    ```
    type.GetTypeInfo().GetCustomAttribute()
    ```
- 引用缓存

    原来是这样：
    
    引用 `System.Runtime.Caching`

    定义 `ObjectCache cache = MemoryCache.Default;` 之后，就可以使用了

    现在是这样：

    在 project.json 中，依赖 `Microsoft.Extensions.Caching.Memory`

    在 Startup.cs 中，添加缓存服务
    ```
    public class Startup{
        public void ConfigureServices(IServiceCollection services){
            services.AddMemoryCache();
        }
    }
    ```
    然后在需要的地方，定义
    ```
    IMemoryCache cache = new MemoryCache(new MemoryCacheOptions());
    ```
- 启用 Session
    原来是这样：

    在纯粹的 ASP.NET 应用中，如果 Handler 需要用到 Session，需要实现 `IRequiresSessionState` 接口

    其他的场景我就不知道了，233333，因为没怎么用过，或者太久了给忘了，2333333

    现在是这样：

    需要注意的是，Session 依赖 Caching，所以除了配置 Session 还要配置 Caching

    在 project.json 中，依赖 `Microsoft.Extensions.Caching.Memory`，`Microsoft.AspNet.Session`

    注册服务
    ```
    public class Startup{
        public void ConfigureServices(IServiceCollection services){
            services.AddMemoryCache();
            services.AddSession(/* options go here */);
        }
    }
    ```
    启用 Session
    ```
    public class Startup{
        public void Configure(IApplicationBuilder app)
        {
            app.UseSession();
        }
    }
    ```
- 获取项目根路径(抄自[链接](https://blog.mariusschulz.com/2016/05/22/getting-the-web-root-path-and-the-content-root-path-in-asp-net-core))

    直接上代码，有什么不明白，看上面的链接
    ```
    // Classic ASP.NET
    public class HomeController : Controller
    {
        public ActionResult Index()
        {
            string physicalWebRootPath = Server.MapPath("~/");
            
            return Content(physicalWebRootPath);
        }
    }
    ```

    ```
    using Microsoft.AspNetCore.Hosting;
    using Microsoft.AspNetCore.Mvc;

    namespace AspNetCorePathMapping
    {
        public class HomeController : Controller
        {
            private readonly IHostingEnvironment _hostingEnvironment;

            public HomeController(IHostingEnvironment hostingEnvironment)
            {
                _hostingEnvironment = hostingEnvironment;
            }

            public ActionResult Index()
            {
                string webRootPath = _hostingEnvironment.WebRootPath;
                string contentRootPath = _hostingEnvironment.ContentRootPath;

                return Content(webRootPath + "\n" + contentRootPath);
            }
        }
    }
    ```