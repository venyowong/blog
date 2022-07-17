---
title: Orleans.Http 介绍
date: 2020-04-19
tags: 
  - .net core
  - rss
  - orleans
---

在前面的博文中有推荐过一款RSS 阅读器 [Inoreader](https://www.inoreader.com/dashboard)，但是前不久突然发现网站打不开了，不清楚什么情况，但大概应该就是闭站了吧，毕竟我以前也遇到过一次 RSS 阅读器突然闭站的情况...跟以前一样，既然市面上没有好用的 RSS 阅读器，就自己动手吧。之前就已经用 UWP 实现过一个客户端了，但是对于那个项目不是特别喜欢，因为对 UWP 不是很熟悉，做出来的效果和体验都比较差。于是想着重新立项，刚好最近看到了一个项目 [Orleans.Http](https://github.com/OrleansContrib/Orleans.Http)，这个项目允许 Orleans 的服务可以通过 web api 的方式直接暴露出来，将 web api 与 Silo 结合在一起，简化了 Orleans 应用的项目结构。在此之前，我也一直想要用 Orleans 做一些实践，但是 Orleans 服务并无法直接调用，所以一直不好着手。

由于 Orleans.Http 本质上是在同一个项目运行 Silo 以及 api，调用 api 接口后，程序会调用 ClusterClient.GetGrain 获取 Grain 实例，并调用对应的接口。所以，在 Orleans.Http 应用中，需要注册 Orleans 服务以及 web api 接口，以下是我在 [Resader](https://github.com/venyowong/resader) 项目中的启动代码：
```
public static IHostBuilder CreateHostBuilder(string[] args)
{
    var config = new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("appsettings.json", optional: true)
        .Build();

    return Microsoft.Extensions.Hosting.Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseUrls(config["Urls"].Split(';').ToArray());
            webBuilder.UseStartup<Startup>();
        })
        .UseOrleans(siloBuilder =>
        {
            siloBuilder.Configure<ClusterOptions>(options =>
            {
                options.ClusterId = "resader";
                options.ServiceId = "resader";
            })
            .AddIncomingGrainCallFilter<ExceptionFilter>()
            .UseAdoNetClustering(options =>
            {
                options.Invariant = "MySql.Data.MySqlClient";
                options.ConnectionString = config["MySql:ConnectionString"];
            })
            .UseAdoNetReminderService(options => 
            {
                options.ConnectionString = config["MySql:ConnectionString"];
                options.Invariant = "MySql.Data.MySqlClient";
            })
            .ConfigureApplicationParts(parts =>
            {
                parts.AddApplicationPart(typeof(UserGrain).Assembly)
                    .AddApplicationPart(typeof(RssGrain).Assembly)
                    .WithReferences();
            })
            .ConfigureEndpoints(siloPort: 7854, gatewayPort: 5489);
        });
}
```
```
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }
    
    // This method gets called by the runtime. Use this method to add services to the container.
    // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddOptions()
            .Configure<Configuration>(this.Configuration);

        services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
        services.AddTransient<RssFetcher>();
        services.AddTransient<IDbConnection>(_ =>
        {
            var connection = new MySqlConnection(this.Configuration["MySql:ConnectionString"]);
            return connection;
        });

        services.AddAuthentication(opt =>
        {
            opt.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            opt.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(opt =>
        {
            opt.RequireHttpsMetadata = false;
            opt.SaveToken = true;
            opt.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(this.Configuration["Jwt:Secret"])),
                ValidateIssuer = false,
                ValidateAudience = false
            };
        });
        services.AddAuthorization();

        services.AddGrainRouter()
            .AddJsonMediaType();
        services.AddScheduler();
        services.AddCors(o => o.AddPolicy("Default", builder =>
        {
            builder.AllowAnyOrigin()
                .AllowAnyMethod()
                .AllowAnyHeader();
        }));
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory, 
        ILogger<IScheduler> schedulerLogger)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        NiologManager.DefaultWriters = new ILogWriter[]
        {
            new ConsoleLogWriter(),
            new FileLogWriter(this.Configuration["Niolog:File"], 10)
        };
        
        loggerFactory.AddProvider(new LoggerProvider());

        var provider = app.ApplicationServices;
        provider.UseScheduler(scheduler =>
        {
            scheduler.Schedule<RssFetcher>()
                .EveryFiveMinutes()
                .PreventOverlapping("RssFetcher");
        })
        .LogScheduledTaskProgress(schedulerLogger)
        .OnError(e =>
        {
            var logger = NiologManager.CreateLogger();
            logger.Warn()
                .Message("Something goes wrong...")
                .Exception(e, true)
                .Write();
        });

        var defaultFile = new DefaultFilesOptions();  
        defaultFile.DefaultFileNames.Clear();  
        defaultFile.DefaultFileNames.Add("index.html");  
        app.UseDefaultFiles(defaultFile)
            .UseStaticFiles()
            .UseCookiePolicy()
            .UseCors("Default");

        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapGrains("grains");
        });
        app.UseRouteGrainProviders(rgppb =>
        {
            rgppb.RegisterRouteGrainProvider<RandomRouteGrainProvider>(nameof(RandomRouteGrainProvider));
        });

        Utility.MakeDapperMapping("Resader.Host.Models");
    }
}
```
在 Program 中，主要注册 Orleans 服务相关的内容。

Orleans.Http 的路由配置是需要声明在 Grain 接口上的，目前支持的 Attribute 大概只有 HttpGet、HttpPost、HttpDelete、HttpPut、Authorize、FromBody、FromQuery 之类的，相对来说功能属于比较简陋，但 Orleans.Http 的项目代码简洁、清晰，完全可以在此基础上进行二次开发。
```
public interface IUserGrain : IGrainWithStringKey
{
    /// <summary>
    /// 注册
    /// <para>101 用户已存在</para>
    /// </summary>
    /// <param name="mail"></param>
    /// <param name="password"></param>
    /// <returns></returns>
    [HttpPost("user/signup", routeGrainProviderPolicy: nameof(RandomRouteGrainProvider))]
    Task<Result<User>> SignUp([FromBody] SignUpRequest request);

    /// <summary>
    /// 登录
    /// <para>101 用户不存在</para>
    /// <para>102 密码错误</para>
    /// </summary>
    /// <param name="mail"></param>
    /// <param name="password"></param>
    /// <returns></returns>
    [HttpPost("user/login", routeGrainProviderPolicy: nameof(RandomRouteGrainProvider))]
    Task<Result<User>> Login([FromBody] LoginRequest request);

    [Authorize]
    [HttpPost("{grainId}/user/resetpassword")]
    Task<Result> ResetPassword([FromBody] ResetPasswordRequest request);
}
```
以登录接口为例，前端调用代码如下：
```
fetch("./grains/user/login", {
    body: JSON.stringify({
        Mail: app.loginForm.mail,
        Password: md5(app.loginForm.password)
    }),
    method: "POST",
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }
})
```
Orleans.Http 会根据 Content-Type 来判断数据的序列化方式，因此如果在接口调用时，未将 Content-Type 设置为正确格式，将无法获取到预期的返回结果。

Orleans.Http 这个项目就是为 Orleans 应用多带来了一种接口输出方式，因此 Grains 服务也可以按照 Orleans 的调用方式进行调用。而且若有部分接口不想通过 api 的方式暴露，只需要不标注 Orleans.Http 的特性，即可保证接口的安全性了。

最后附上 Resader 项目的线上实例[链接](https://venyo.cn/resader/)