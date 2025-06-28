---
sidebar_position: 20
---

# 20. 定时任务

:::tip 提示

Admin.NET 任务调度集成组件 Sundial, 功能齐全的开源分布式作业调度系统，简单好用。 Cron 表达式解析库 采用 TimeCrontab 支持 Cron 所有特性。编写 Cron 表达式推荐使用在线 Cron 表达式生成器 https://cron.qqe2.com/
:::

创建一个定时任务非常容易，定义作业并实现 IJob 接口即可，不需要手动注册。框架自动注册所有的定时任务并作数据库持久化（将定时任务存到数据库里面）。建议每个定时任务类上面都贴 `JobDetail` 和 `Daily` 作业信息特性。

```csharp
[JobDetail("job_xxx", Description = "xxx", GroupName = "default", Concurrent = false)]
[Daily(TriggerId = "trigger_xxx", Description = "xxx")]
public class MyJob : IJob
{
    private readonly ILogger<MyJob> _logger;
    public MyJob(ILogger<MyJob> logger)
    {
        _logger = logger;
    }

    public Task ExecuteAsync(JobExecutingContext context, CancellationToken stoppingToken)
    {
        _logger.LogInformation(context.ToString());
        return Task.CompletedTask;
    }
}
```

`JobDetail` 任务详情特性描述里面 `Concurrent` 可以指定是否串行还是并行执行。`Trigger` 触发器特性可以是任何 `Cron` 表达式，内置时间周期特性如下：

- [Period(5000)]：毫秒周期（间隔）作业触发器特性
- [PeriodSeconds(5)]：秒周期（间隔）作业触发器特性
- [PeriodMinutes(5)]：分钟周期（间隔）作业触发器特性
- [PeriodHours(5)]：小时周期（间隔）作业触发器特性
- [Cron("* * * * *", CronStringFormat.Default)]：Cron 表达式作业触发器特性
- [Secondly]：每秒开始作业触发器特性
- [Minutely]：每分钟开始作业触发器特性
- [Hourly]：每小时开始作业触发器特性
- [Daily]：每天（午夜）开始作业触发器特性
- [Monthly]：每月 1 号（午夜）开始作业触发器特性
- [Weekly]：每周日（午夜）开始作业触发器特性
- [Yearly]：每年 1 月 1 号（午夜）开始作业触发器特性
- [Workday]：每周一至周五（午夜）开始触发器特性
- [SecondlyAt]：特定秒开始作业触发器特性
- [MinutelyAt]：每分钟特定秒开始作业触发器特性
- [HourlyAt]：每小时特定分钟开始作业触发器特性
- [DailyAt]：每天特定小时开始作业触发器特性
- [MonthlyAt]：每月特定天（午夜）开始作业触发器特性
- [WeeklyAt]：每周特定星期几（午夜）开始作业触发器特性
- [YearlyAt]：每年特定月 1 号（午夜）开始作业触发器特性

下面是系统自带的定时任务示例：

```csharp
namespace Admin.NET.Core;

/// <summary>
/// 清理日志作业任务
/// </summary>
[JobDetail("job_log", Description = "清理操作日志", GroupName = "default", Concurrent = false)]
[Daily(TriggerId = "trigger_log", Description = "清理操作日志")]
public class LogJob : IJob
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger _logger;

    public LogJob(IServiceScopeFactory scopeFactory, ILoggerFactory loggerFactory)
    {
        _scopeFactory = scopeFactory;
        _logger = loggerFactory.CreateLogger(CommonConst.SysLogCategoryName);
    }

    public async Task ExecuteAsync(JobExecutingContext context, CancellationToken stoppingToken)
    {
        using var serviceScope = _scopeFactory.CreateScope();

        var logVisRep = serviceScope.ServiceProvider.GetRequiredService<SqlSugarRepository<SysLogVis>>();
        var logOpRep = serviceScope.ServiceProvider.GetRequiredService<SqlSugarRepository<SysLogOp>>();
        var logDiffRep = serviceScope.ServiceProvider.GetRequiredService<SqlSugarRepository<SysLogDiff>>();

        var daysAgo = 30; // 删除30天以前
        await logVisRep.CopyNew().AsDeleteable().Where(u => (DateTime)u.CreateTime < DateTime.Now.AddDays(-daysAgo)).ExecuteCommandAsync(stoppingToken); // 删除访问日志
        await logOpRep.CopyNew().AsDeleteable().Where(u => (DateTime)u.CreateTime < DateTime.Now.AddDays(-daysAgo)).ExecuteCommandAsync(stoppingToken); // 删除操作日志
        await logDiffRep.CopyNew().AsDeleteable().Where(u => (DateTime)u.CreateTime < DateTime.Now.AddDays(-daysAgo)).ExecuteCommandAsync(stoppingToken); // 删除差异日志

        var originColor = Console.ForegroundColor;
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"【{DateTime.Now}】清理系统日志（30天前）");
        Console.ForegroundColor = originColor;

        // 自定义日志
        _logger.LogInformation($"【{DateTime.Now}】清理系统日志...");
    }
}
```

:::tip 提示

如上面代码所示，若需要在定时任务里面操作数据库，则仓储需要 `CopyNew()` 进行操作，否则会出现线程异常错误。同时，任务内容一般都在当前作用域内进行 `using var serviceScope = _scopeFactory.CreateScope();`。
:::