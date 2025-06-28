---
sidebar_position: 22
---

# 22. 日志记录

:::tip 提示

Admin.NET 已实现包括行为日志、异常日志、审计日志、访问日志等功能。日志采用 `Furion` 自带的日志组件，简单方便。.NET 本身自带的内置日志组件[ILogger](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/logging?tabs=command-line) 也很好，可以通过配置不同的记录提供程序将日志写入不同的目标。 基本日志记录提供程序是内置的，并且还可以使用许多第三方提供程序，比如 log4net。
:::

目前日志存储支持数据库存储、文件存储、ElasticSearch，配置文件 [/Configuration/Logging.json](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Admin.NET/Admin.NET.Application/Configuration/Logging.json)。其中 Monitor.GlobalEnabled 可以配置全局是否启动日志（如若不需要记录日志，可以关闭此项配置）。

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information"
    },
    "File": {
      "Enabled": false, // 启用文件日志
      "FileName": "logs/{0:yyyyMMdd}_{1}.log", // 日志文件
      "Append": true, // 追加覆盖
      // "MinimumLevel": "Information", // 日志级别
      "FileSizeLimitBytes": 10485760, // 10M=10*1024*1024
      "MaxRollingFiles": 30 // 只保留30个文件
    },
    "Database": {
      "Enabled": true, // 启用数据库日志
      "MinimumLevel": "Information"
    },
    "ElasticSearch": {
      "Enabled": false, // 启用ES日志
      "AuthType": "Basic", // ES认证类型，可选 Basic、ApiKey、Base64ApiKey
      "User": "admin", // Basic认证的用户名，使用Basic认证类型时必填
      "Password": "123456", // Basic认证的密码，使用Basic认证类型时必填
      "ApiId": "", // 使用ApiKey认证类型时必填
      "ApiKey": "", // 使用ApiKey认证类型时必填
      "Base64ApiKey": "TmtrOEszNEJuQ0NyaWlydGtROFk6SG1RZ0w3YzBTc2lCanJTYlV3aXNzZw==", // 使用Base64ApiKey认证类型时必填
      "Fingerprint": "37:08:6A:C6:06:CC:9A:43:CF:ED:25:A2:1C:A4:69:57:90:31:2C:06:CA:61:56:39:6A:9C:46:11:BD:22:51:DA", // ES使用Https时的证书指纹
      "ServerUris": [ "http://192.168.1.100:9200" ], // 地址
      "DefaultIndex": "adminnet" // 索引
    },
    "Monitor": {
      "GlobalEnabled": true, // 启用全局拦截日志
      "IncludeOfMethods": [], // 拦截特定方法，当GlobalEnabled=false有效
      "ExcludeOfMethods": [], // 排除特定方法，当GlobalEnabled=true有效
      "BahLogLevel": "Information", // Oops.Oh 和 Oops.Bah 业务日志输出级别
      "WithReturnValue": true, // 是否包含返回值，默认true
      "ReturnValueThreshold": 500, // 返回值字符串阈值，默认0全量输出
      "JsonBehavior": "None", // 是否输出Json，默认None(OnlyJson、All)
      "JsonIndented": false, // 是否格式化Json
      "UseUtcTimestamp": false, // 时间格式UTC、LOCAL
      "ConsoleLog": true // 是否显示控制台日志
    }
  }
}
```

系统集成实现类 [Admin.NET.Core/Logging/LoggingSetup.cs](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Admin.NET/Admin.NET.Core/Logging/LoggingSetup.cs) 供其参考，都已经自动化，只需要改改配置文件即可，无需写额外的代码逻辑实现。

```csharp
namespace Admin.NET.Core;

public static class LoggingSetup
{
    /// <summary>
    /// 日志注册
    /// </summary>
    /// <param name="services"></param>
    public static void AddLoggingSetup(this IServiceCollection services)
    {
        // 日志监听
        services.AddMonitorLogging(options =>
        {
            options.IgnorePropertyNames = new[] { "Byte" };
            options.IgnorePropertyTypes = new[] { typeof(byte[]) };
        });

        // 控制台日志
        var consoleLog = App.GetConfig<bool>("Logging:Monitor:ConsoleLog", true);
        services.AddConsoleFormatter(options =>
        {
            options.DateFormat = "yyyy-MM-dd HH:mm:ss(zzz) dddd";
            //options.WithTraceId = true; // 显示线程Id
            //options.WithStackFrame = true; // 显示程序集
            options.WriteFilter = (logMsg) =>
            {
                return consoleLog;
            };
        });

        // 日志写入文件
        if (App.GetConfig<bool>("Logging:File:Enabled", true))
        {
            var loggingMonitorSettings = App.GetConfig<LoggingMonitorSettings>("Logging:Monitor", true);
            Array.ForEach(new[] { LogLevel.Information, LogLevel.Warning, LogLevel.Error }, logLevel =>
            {
                services.AddFileLogging(options =>
                {
                    options.WithTraceId = true; // 显示线程Id
                    options.WithStackFrame = true; // 显示程序集
                    options.FileNameRule = fileName => string.Format(fileName, DateTime.Now, logLevel.ToString()); // 每天创建一个文件
                    options.WriteFilter = logMsg => logMsg.LogLevel == logLevel; // 日志级别
                    options.HandleWriteError = (writeError) => // 写入失败时启用备用文件
                    {
                        writeError.UseRollbackFileName(Path.GetFileNameWithoutExtension(writeError.CurrentFileName) + "-oops" + Path.GetExtension(writeError.CurrentFileName));
                    };
                    if (loggingMonitorSettings.JsonBehavior == JsonBehavior.OnlyJson)
                    {
                        options.MessageFormat = LoggerFormatter.Json;
                        // options.MessageFormat = LoggerFormatter.JsonIndented;
                        options.MessageFormat = (logMsg) =>
                        {
                            var jsonString = logMsg.Context.Get("loggingMonitor");
                            return jsonString?.ToString();
                        };
                    }
                });
            });
        }

        // 日志写入ElasticSearch
        if (App.GetConfig<bool>("Logging:ElasticSearch:Enabled", true))
        {
            services.AddDatabaseLogging<ElasticSearchLoggingWriter>(options =>
            {
                options.WithTraceId = true; // 显示线程Id
                options.WithStackFrame = true; // 显示程序集
                options.IgnoreReferenceLoop = false; // 忽略循环检测
                options.MessageFormat = LoggerFormatter.Json;
                options.WriteFilter = (logMsg) =>
                {
                    return logMsg.LogName == CommonConst.SysLogCategoryName; // 只写LoggingMonitor日志
                };
            });
        }

        // 日志写入数据库
        if (App.GetConfig<bool>("Logging:Database:Enabled", true))
        {
            services.AddDatabaseLogging<DatabaseLoggingWriter>(options =>
            {
                options.WithTraceId = true; // 显示线程Id
                options.WithStackFrame = true; // 显示程序集
                options.IgnoreReferenceLoop = false; // 忽略循环检测
                options.WriteFilter = (logMsg) =>
                {
                    return logMsg.LogName == CommonConst.SysLogCategoryName; // 只写LoggingMonitor日志
                };
            });
        }
    }
}
```

1、泛型方式：通过注入 `ILogger<T>` 对象进行操作日志，如

```csharp
public class xxxTest
{
    private readonly ILogger<xxxTest> _logger;

    public xxxTest(ILogger<xxxTest> logger)
    {
        _logger = logger;
    }

    public void Test()
    {
        _logger.LogInformation("日志内容。。。");
    }
}
```

2、工厂方式：`ILoggerFactory` 工厂方式

```csharp
public class xxxTest
{
    private readonly ILogger _logger;

    public xxxTest(ILoggerFactory logger)
    {
        _logger = logger.CreateLogger("我的分组");
    }

    public void Test()
    {
        _logger.LogInformation("日志内容。。。");
    }
}
```

3、静态方式：Log 静态类方式

```csharp
// 创建日志对象
var logger = Log.CreateLogger("日志名称");

// 创建日志工厂
using var loggerFactory = Log.CreateLoggerFactory(builder => {

});

// 日志记录
Log.Information("xxx");
Log.Warning("xxx");
Log.Error("xxx");
Log.Debug("xxx");
Log.Trace("xxx");
Log.Critical("xxx");
```

4、字符串拓展的方式

```csharp
"xxx".LogInformation();

"xxx".LogError<xxxController>();

"xxx".SetCategory<xxxController>().LogInformation();
```

:::tip 提示

内置了日志分组常量 `CommonConst.SysLogCategoryName`，在需要自定义日志且与系统匹配起来自动存储时，`_logger = loggerFactory.CreateLogger(CommonConst.SysLogCategoryName);` 这时自定义的日志就会自动存储。
:::

下面时自定义日志存储的实现：

```csharp
    public async Task WriteAsync(LogMessage logMsg, bool flush)
    {
        var jsonStr = logMsg.Context?.Get("loggingMonitor")?.ToString();
        if (string.IsNullOrWhiteSpace(jsonStr))
        {
            await _db.Insertable(new SysLogOp
            {
                DisplayTitle = "自定义操作日志",
                LogDateTime = logMsg.LogDateTime,
                EventId = logMsg.EventId.Id,
                ThreadId = logMsg.ThreadId,
                TraceId = logMsg.TraceId,
                Exception = logMsg.Exception == null ? null : JSON.Serialize(logMsg.Exception),
                Message = logMsg.Message,
                LogLevel = logMsg.LogLevel,
                Status = "200",
            }).ExecuteCommandAsync();
            return;
        }
    }
```