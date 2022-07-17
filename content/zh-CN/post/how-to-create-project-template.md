---
title: 创建自定义项目模板
date: 2022-01-04
tags:
  - .net
---

创建一个集成了所有自己常用组件的项目模板，能够在新项目启动时节省大量的时间。.net对自建项目模板支持还是比较友好的，但问题是文档不多，我查到的文档里面较少比较详细的就只有[这篇博客](https://devblogs.microsoft.com/dotnet/how-to-create-your-own-templates-for-dotnet-new/)。接下来，我会按照步骤重新创建一个模板来解释模板文件的配置。

1. 创建一个控制台项目(也可以是其他类型的项目)
   ```
   dotnet new console -n TestTemplate
   ```
2. 在项目根目录下创建 .template.config 目录，并在该目录下创建 template.json 文件
   ```
    {
        "$schema": "http://json.schemastore.org/template",
        "author": "Venyo Wong",
        "classifications": [ "Web", "MVC" ],
        "identity": "Base",
        "name": "Base",
        "shortName": "base",
        "preferNameDirectory": true,
        "symbols": {
            "name": {
                "type": "parameter",
                "replaces": "TestTemplate",
                "FileRename": "TestTemplate",
                "isRequired": true
            }
        }
    }
   ```
   这样就创建好了一个最基础的项目模板。
   - classifications 字段代表该模板的类型，会在 dotnet new 的模板列表体现出来
   - identity、name、shortName 是用来识别模板的字段
   - preferNameDirectory 该字段表示是否将目录名称作为项目名称，比如该字段设置为 true，在一个名为 Test 的目录下创建项目，在未指定项目名称的前提下，项目名称默认为 Test
   - symbols 代表创建项目时可以指定的参数
   - symbols.name 表示使用此模板创建项目时可以指定 name 参数
   - symbols.name.replaces: "TestTemplate" 表示在创建项目时，需要把文件中的 TestTemplate 字符串替换为用户传入的 name 参数
   - symbols.name.FileRename: "TestTemplate" 表示在创建项目时，需要把文件名中的 TestTemplate 字符串替换为用户传入的 name 参数
3. 修改项目内容，此处仅在 Program.cs 文件中加入 TestTemplate，用于测试效果
   ```
    // See https://aka.ms/new-console-template for more information
    Console.WriteLine("TestTemplate");
   ```
4. 安装本地项目模板
   ```
    >dotnet new --install.

    将安装以下模板包:
    D:\code\test\TestTemplate

    成功: D:\code\test\TestTemplate 已安装以下模板:
    模板名   短名称   语言  标记
    ----  ----  --  -------
    Base  base      Web/MVC
   ```
5. 使用自定义模板创建项目
   ```
    >dotnet new base -n TestProject

    已成功创建模板“Base”。
   ```
   打开 TestProject 项目可以看到项目配置文件名变更为 TestProject.csproj，Program.cs 文件内容变更为
   ```
    // See https://aka.ms/new-console-template for more information
    Console.WriteLine("TestProject");
   ```
6. 排除部分文件、目录，在项目模板开发过程中，难免会自动生成一些文件，比如 .vs、.vscode 等，想要将这些文件、目录排除在模板之外，需要做额外的配置。
   ```
    {
        "$schema": "http://json.schemastore.org/template",
        "author": "Venyo Wong",
        "classifications": [ "Web", "MVC" ],
        "identity": "Base",
        "name": "Base",
        "shortName": "base",
        "preferNameDirectory": true,
        "symbols": {
            "name": {
                "type": "parameter",
                "replaces": "TestTemplate",
                "FileRename": "TestTemplate",
                "isRequired": true
            }
        },
        "sources": [
            {
                "modifiers": [
                    {
                        "exclude": [
                            ".template.config/**",
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
        ]
    }
   ```
7. 添加可选模块，此处以 Newtonsoft.Json 为例进行说明。
   
   项目配置引用 Newtonsoft.Json 包。
   ```
    <ItemGroup Condition="$(json)">
        <PackageReference Include="Newtonsoft.Json" Version="13.0.1" />
    </ItemGroup>
   ```
   
   在项目根目录添加 Json 目录，并且添加 JsonHelper.cs 文件，内容如下：
   ```
    using Newtonsoft.Json;

    namespace TestTemplate.Json;

    public static class JsonHelper
    {
        public static (string, Exception?) ToJson(this object obj)
        {
            if (obj == null)
            {
                return default;
            }

            try
            {
                return (JsonConvert.SerializeObject(obj), null);
            }
            catch (Exception ex)
            {
                return (string.Empty, ex);
            }
        }
    }
   ```
   Program.cs 文件修改为：
   ```
    // See https://aka.ms/new-console-template for more information
    #if (!json)
    Console.WriteLine("TestTemplate");
    #else
    using TestTemplate.Json;

    Console.WriteLine(JsonHelper.ToJson(new { Key = "Value" }));
    #endif
   ```
   最后修改 template.json 配置如下：
   ```
    {
        "$schema": "http://json.schemastore.org/template",
        "author": "Venyo Wong",
        "classifications": [ "Web", "MVC" ],
        "identity": "Base",
        "name": "Base",
        "shortName": "base",
        "preferNameDirectory": true,
        "symbols": {
            "name": {
                "type": "parameter",
                "replaces": "TestTemplate",
                "FileRename": "TestTemplate",
                "isRequired": true
            },
            "json": {
                "type": "parameter", 
                "dataType":"bool", 
                "defaultValue": "true"
            }
        },
        "sources": [
            {
                "modifiers": [
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
                    },
                    {
                        "condition": "(!json)",
                        "exclude": [
                            "Json/**"
                        ]
                    }
                ]
            }
        ]
    }
   ```
8. 生成带有 json 模块的项目
   ```
    dotnet new base -n TestProject

    生成项目结构如下：
    TestProject
    │  Program.cs
    │  TestProject.csproj
    │
    └─Json
            JsonHelper.cs
   ```
9. 生成不带有 json 模块的项目
    ```
    dotnet new base -n TestProject --json false

    生成项目结构如下：
    TestProject
        Program.cs
        TestProject.csproj

    没有子文件夹
    ```

以上为我自己常用的模板配置，其他还有一些可配置项，感兴趣的读者可以查阅官方文档或者查看官方demo。欢迎联系讨论：venyowong@163.com。