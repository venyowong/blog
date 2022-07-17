---
title: .net core 项目模板介绍
date: 2020-08-06
tags:
  - .net core
---

在上家公司，我主要是负责一个交易系统的迭代和维护，大部分工作时间也都是在同一个项目里写代码。但是换了工作之后，因为工作性质的变化，我变得需要创建维护多个项目，大概就是体量上的差距吧。经常创建新项目，在搭建项目基础结构时，需要写大量重复的代码，虽然大部分代码只需要复制粘贴，然后作少量修改即可，比如类文件拷贝之后，修改个命名空间。但是这样的重复劳动不仅没有意义，而且特别浪费时间，因此我需要学习一个方法，能够让自己创建项目时，只需要指定一下项目名称，就可以得到完整的基础项目，就像 `dotnet new console -n Test`。

幸好 `dotnet new` 命令有这方面的支持，官方文档可点击[传送门](https://docs.microsoft.com/zh-cn/dotnet/core/tools/custom-templates)访问。

浏览官方文档就可以了解到，其实项目模板就是在普通项目的基础上多加了一个配置：在项目根目录创建 `.template.config` 目录，在此目录下创建 `template.json` 配置文件。以我个人创建的一个[项目模板](https://github.com/venyowong/ProjectTemplate)配置文件为例：
```
{
    "$schema": "http://json.schemastore.org/template",
    "author": "Venyo Wong",
    "classifications": [ "Web", "MVC" ],
    "identity": "Base",
    "name": "Base",
    "shortName": "base",
    "symbols": {
        "ProjectName": {
            "type": "parameter",
            "replaces": "ProjectTemplate",
            "FileRename": "ProjectTemplate",
            "isRequired": true
        }
    },
    "sources": [
        {
            "exclude": [
                ".template.config/**/*",
                ".vs/**",
                ".vscode/**",
                "bin/**",
                "log/**",
                "obj/**",
                "ProjectTemplate.xml"
            ]
        }
    ]
}
```

`classifications identity name shortName` 这几个参数主要对应于 `dotnet new --list` 命令展示的信息，根据项目情况填写即可。

`symbols` 看了网上的文章，大多用于为模板添加参数，但是它有一个 `type` 属性，可以看出它的功能应该不仅如此。但我个人也只用到了参数功能，所以没深入了解，在这里附上两个参考链接，感兴趣可以浏览看是否能获得有用的信息，[template.json schema](http://json.schemastore.org/template) [dotnet templating](https://github.com/dotnet/templating)。

`replaces FileRename` 表示用户输入的参数用于覆盖哪个字符串，通过这两个参数就可以自由指定项目命名空间了。

`sources.exclude` 这个配置顾名思义就是创建项目时，排除哪些文件，将新项目非必需的文件排除掉能使创建出来的项目保持整洁，不需要做二次整理。

当项目准备就绪，配置文件也写好后，在项目根目录执行 `dotnet new --install .` 就可将项目模板安装到本地，之后执行 `dotnet new base --ProjectName Test -o Test` 就可以创建出个人定制的基础项目结构了。

最后，稍微介绍一下个人创建的[项目模板](https://github.com/venyowong/ProjectTemplate)，该项目模板集成了 Swagger、AspNetCoreRateLimit、Polly、Dapper、StackExchange.Redis、Serilog、MySql.Data，这些组件的集成方式都比较宽松，便于"拆卸"：先把 csproj 文件中的相关配置删除，然后将 IDE 报错的代码删除即可。