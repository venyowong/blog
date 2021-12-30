---
title: 使用 Nlog 将日志打印到 Logstash 的监控接口
date: 2020-01-28 12:36:21
tags: 
  - .net
  - nlog 
  - logstash
---

Logstash提供了多种监听日志打印的方式，而Nlog也提供了多种输出日志的方式，当Nlog的输出配置与Logstash的输入配置相对应，就能够让Nlog打印出来的日志都存储到Elasticsearch中

以下介绍3种配置方式：

1. 文件

    Logstash:
    ```
    input {
        file {
            path => "D:/Log/Application/*Log.txt"
            type => "Application"
        }
    }
    ```

    NLog:
    ```
    <targets>
        <target xsi:type="File" name="file" filename="D:/Log/Application/${shortdate}Log.txt" layout="${longdate} ${uppercase:${level}} ${message}"/>
    </targets>
    ```

2. tcp
    Logstash:
    ```
    input {
        tcp{
           port => 8001
           type => "TcpLog"
        }
    }
    ```

    NLog:
    ```
    <targets>
        <target xsi:type="Network" name="tcp" address="tcp://127.0.0.1:8001" layout="${longdate} ${uppercase:${level}} ${message}"/>
    </targets>
    ```

3. udp
    Logstash:
    ```
    input {
        udp{
           port => 8002
           type => "UdpLog"
        }
    }
    ```

    NLog:
    ```
    <targets>
        <target xsi:type="Network" name="udp" address="udp://127.0.0.1:8001" layout="${longdate} ${uppercase:${level}} ${message}"/>
    </targets>
    ```