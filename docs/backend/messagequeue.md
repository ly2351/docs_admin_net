---
sidebar_position: 19
---

# 19. 消息队列

:::tip 提示

消息队列顾名思义就是一个存放消息的队列。最简单的消息队列包含 3 个角色：

- 生产者：将消息存入队列中
- 队列：存放和管理消息
- 消费者： 将消息从队列中取出来并做业务处理。

Admin.NET 已经集成事件总线和消息队列。事件总线是对发布-订阅模式的一种实现。它是一种集中式事件处理机制，允许不同的组件之间进行彼此通信而又不需要相互依赖，达到一种解耦的目的。
:::

## 事件总线
事件总线通常用于耗时、不用及时反馈等场景，比如登录的时候发短信、发邮件、通知等等，不需要立刻通知，后台处理即可。定义事件订阅者 `ToDoEventSubscriber`

```csharp
public class xxxSubscriber : IEventSubscriber
{
    private readonly ILogger<xxxSubscriber> _logger;
    public xxxSubscriber(ILogger<xxxSubscriber> logger)
    {
        _logger = logger;
    }

    [EventSubscribe("ToDo:Add")]
    public async Task AddToDo(EventHandlerExecutingContext context)
    {
        var todo = context.Source;
        _logger.LogInformation("事件总线：{Name}", todo.Payload);
        await Task.CompletedTask;
    }
}
```

下面是框架事件订阅的实现示例 `Admin.NET.Core/EventBus/AppEventSubscriber.cs`

```csharp
namespace Admin.NET.Core;

/// <summary>
/// 事件订阅
/// </summary>
public class AppEventSubscriber : IEventSubscriber, ISingleton, IDisposable
{
    private readonly IServiceScope _serviceScope;

    public AppEventSubscriber(IServiceScopeFactory scopeFactory)
    {
        _serviceScope = scopeFactory.CreateScope();
    }

    /// <summary>
    /// 增加异常日志
    /// </summary>
    /// <param name="context"></param>
    /// <returns></returns>
    [EventSubscribe(CommonConst.AddExLog)]
    public async Task CreateExLog(EventHandlerExecutingContext context)
    {
        var rep = _serviceScope.ServiceProvider.GetRequiredService<SqlSugarRepository<SysLogEx>>();
        await rep.InsertAsync((SysLogEx)context.Source.Payload);
    }

    /// <summary>
    /// 发送异常邮件
    /// </summary>
    /// <param name="context"></param>
    /// <returns></returns>
    [EventSubscribe(CommonConst.SendErrorMail)]
    public async Task SendOrderErrorMail(EventHandlerExecutingContext context)
    {
        //var mailTempPath = Path.Combine(App.WebHostEnvironment.WebRootPath, "Temp\\ErrorMail.tp");
        //var mailTemp = File.ReadAllText(mailTempPath);
        //var mail = await _serviceScope.ServiceProvider.GetRequiredService<IViewEngine>().RunCompileFromCachedAsync(mailTemp, );

        var title = "Admin.NET 系统异常";
        await _serviceScope.ServiceProvider.GetRequiredService<SysEmailService>().SendEmail(JSON.Serialize(context.Source.Payload), title);
    }

    /// <summary>
    /// 释放服务作用域
    /// </summary>
    public void Dispose()
    {
        _serviceScope.Dispose();
    }
}
```

依赖注入 `IEventPublisher` 服务，指定事件订阅名称即可发送/消费数据

```csharp
public class xxxController : ControllerBase
{
    private readonly IEventPublisher _eventPublisher;

    public xxxController(IEventPublisher eventPublisher)
    {
        _eventPublisher = eventPublisher;
    }

    // 发布 ToDo:Add 消息
    public async Task AddDoTo(string name)
    {
        await _eventPublisher.PublishAsync(new ChannelEventSource("ToDo:Add", name));
        
        // await _eventPublisher.PublishDelayAsync(new ChannelEventSource("ToDo:Add", name), 5000); // 延迟 5s
    }
}
```

:::tip 提示

所有自定义的事件订阅者只需要实现/继承 `ISingleton` 接口，无需手动注册服务，框架已通过 `services.AddEventBus()` 全局注册所有。
:::

## 自定义事件源存储器

:::tip 提示

事件总线默认采用 Channel 作为事件源 IEventSource 存储器，可使用任何消息队列组件进行替换，如 Kafka、RabbitMQ、Redis 等。Admin.NET 已经集成 `RabbitMQ` 和 `Redis` 自定义事件源存储器。

## RabbitMQ 事件源存储器

具体 `RabbitMQ` 事件源实现如下, 配置文件 Configuration/EventBus.json

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "EventBus": {
    "RabbitMQ": {
      "UserName": "adminnet",
      "Password": "adminnet++123456",
      "HostName": "127.0.0.1",
      "Port": 5672
    }
  }
}
```

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Threading.Channels;

namespace Admin.NET.Core;

/// <summary>
/// RabbitMQ自定义事件源存储器
/// </summary>
public class RabbitMQEventSourceStore : IEventSourceStorer
{
    /// <summary>
    /// 内存通道事件源存储器
    /// </summary>
    private readonly Channel<IEventSource> _channel;

    /// <summary>
    /// 通道对象
    /// </summary>
    private readonly IModel _model;

    /// <summary>
    /// 连接对象
    /// </summary>
    private readonly IConnection _connection;

    /// <summary>
    /// 路由键
    /// </summary>
    private readonly string _routeKey;

    /// <summary>
    /// 构造函数
    /// </summary>
    /// <param name="factory">连接工厂</param>
    /// <param name="routeKey">路由键</param>
    /// <param name="capacity">存储器最多能够处理多少消息，超过该容量进入等待写入</param>
    public RabbitMQEventSourceStore(ConnectionFactory factory, string routeKey, int capacity)
    {
        // 配置通道，设置超出默认容量后进入等待
        var boundedChannelOptions = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };

        // 创建有限容量通道
        _channel = Channel.CreateBounded<IEventSource>(boundedChannelOptions);

        // 创建连接
        _connection = factory.CreateConnection();
        _routeKey = routeKey;

        // 创建通道
        _model = _connection.CreateModel();

        // 声明路由队列
        _model.QueueDeclare(routeKey, false, false, false, null);

        // 创建消息订阅者
        var consumer = new EventingBasicConsumer(_model);

        // 订阅消息并写入内存 Channel
        consumer.Received += (ch, ea) =>
        {
            // 读取原始消息
            var stringEventSource = Encoding.UTF8.GetString(ea.Body.ToArray());

            // 转换为 IEventSource，如果自定义了 EventSource，注意属性是可读可写
            var eventSource = JSON.Deserialize<ChannelEventSource>(stringEventSource);

            // 写入内存管道存储器
            _channel.Writer.WriteAsync(eventSource);

            // 确认该消息已被消费
            _model.BasicAck(ea.DeliveryTag, false);
        };

        // 启动消费者且设置为手动应答消息
        _model.BasicConsume(routeKey, false, consumer);
    }

    /// <summary>
    /// 将事件源写入存储器
    /// </summary>
    /// <param name="eventSource">事件源对象</param>
    /// <param name="cancellationToken">取消任务 Token</param>
    /// <returns><see cref="ValueTask"/></returns>
    public async ValueTask WriteAsync(IEventSource eventSource, CancellationToken cancellationToken)
    {
        if (eventSource == default)
            throw new ArgumentNullException(nameof(eventSource));

        // 判断是否是 ChannelEventSource 或自定义的 EventSource
        if (eventSource is ChannelEventSource source)
        {
            // 序列化及发布
            var data = Encoding.UTF8.GetBytes(JSON.Serialize(source));
            _model.BasicPublish("", _routeKey, null, data);
        }
        else
        {
            // 处理动态订阅
            await _channel.Writer.WriteAsync(eventSource, cancellationToken);
        }
    }

    /// <summary>
    /// 从存储器中读取一条事件源
    /// </summary>
    /// <param name="cancellationToken">取消任务 Token</param>
    /// <returns>事件源对象</returns>
    public async ValueTask<IEventSource> ReadAsync(CancellationToken cancellationToken)
    {
        var eventSource = await _channel.Reader.ReadAsync(cancellationToken);
        return eventSource;
    }

    /// <summary>
    /// 释放非托管资源
    /// </summary>
    public void Dispose()
    {
        _model.Dispose();
        _connection.Dispose();
    }
}
```

## Redis 事件源存储器

具体 `Redis` 事件源实现如下, 配置文件 Configuration/Cache.json

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "Cache": {
    "Prefix": "adminnet_", // 全局缓存前缀
    "CacheType": "Memory", // Memory、Redis
    "Redis": {
      "Configuration": "server=127.0.0.1:6379;password=;db=5;", // Redis连接字符串
      "Prefix": "adminnet_", // Redis前缀（目前没用）
      "MaxMessageSize": "1048576" // 最大消息大小 默认1024 * 1024
    }
  },
  "Cluster": { // 集群配置
    "Enabled": false, // 启用集群：前提开启Redis缓存模式
    "ServerId": "adminnet", // 服务器标识
    "ServerIp": "", // 服务器IP
    "SignalR": {
      "RedisConfiguration": "127.0.0.1:6379,ssl=false,password=,defaultDatabase=5",
      "ChannelPrefix": "signalrPrefix_"
    },
    "DataProtecteKey": "AdminNet:DataProtection-Keys",
    "IsSentinel": false, // 是否哨兵模式
    "SentinelConfig": {
      "DefaultDb": "4",
      "EndPoints": [ // 哨兵端口
        // "10.10.0.124:26380"
      ],
      "MainPrefix": "adminNet:",
      "Password": "123456",
      "SentinelPassword": "adminNet",
      "ServiceName": "adminNet",
      "SignalRChannelPrefix": "signalR:"
    }
  }
}
```

```csharp
using System.Threading.Channels;

namespace Admin.NET.Core;

/// <summary>
/// Redis自定义事件源存储器
/// </summary>
public sealed class RedisEventSourceStorer : IEventSourceStorer, IDisposable
{
    /// <summary>
    /// 消费者
    /// </summary>
    private readonly EventConsumer<ChannelEventSource> _eventConsumer;

    /// <summary>
    /// 内存通道事件源存储器
    /// </summary>
    private readonly Channel<IEventSource> _channel;

    /// <summary>
    /// Redis 连接对象
    /// </summary>
    private readonly FullRedis _redis;

    /// <summary>
    /// 路由键
    /// </summary>
    private readonly string _routeKey;

    /// <summary>
    /// 构造函数
    /// </summary>
    /// <param name="redis">Redis 连接对象</param>
    /// <param name="routeKey">路由键</param>
    /// <param name="capacity">存储器最多能够处理多少消息，超过该容量进入等待写入</param>
    public RedisEventSourceStorer(ICache redis, string routeKey, int capacity)
    {
        // 配置通道，设置超出默认容量后进入等待
        var boundedChannelOptions = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };

        // 创建有限容量通道
        _channel = Channel.CreateBounded<IEventSource>(boundedChannelOptions);

        _redis = redis as FullRedis;
        _routeKey = routeKey;

        // 创建消息订阅者
        _eventConsumer = new EventConsumer<ChannelEventSource>(_redis, _routeKey);

        // 订阅消息写入 Channel
        _eventConsumer.Received += (send, cr) =>
        {
            // 反序列化消息
            //var eventSource = JsonConvert.DeserializeObject<ChannelEventSource>(cr);

            // 写入内存管道存储器
            Task.Run(async () =>
            {
                await _channel.Writer.WriteAsync(cr);
            });
        };

        // 启动消费者
        _eventConsumer.Start();
    }

    /// <summary>
    /// 将事件源写入存储器
    /// </summary>
    /// <param name="eventSource">事件源对象</param>
    /// <param name="cancellationToken">取消任务 Token</param>
    /// <returns><see cref="ValueTask"/></returns>
    public async ValueTask WriteAsync(IEventSource eventSource, CancellationToken cancellationToken)
    {
        // 空检查
        if (eventSource == default)
        {
            throw new ArgumentNullException(nameof(eventSource));
        }

        // 这里判断是否是 ChannelEventSource 或者 自定义的 EventSource
        if (eventSource is ChannelEventSource source)
        {
            // 序列化消息
            //var data = JsonSerializer.Serialize(source);

            // 获取一个订阅对象
            var queue = _redis.GetQueue<ChannelEventSource>(_routeKey);

            // 异步发布
            await Task.Factory.StartNew(() =>
            {
                queue.Add(source);
            }, cancellationToken, TaskCreationOptions.LongRunning, System.Threading.Tasks.TaskScheduler.Default);
        }
        else
        {
            // 这里处理动态订阅问题
            await _channel.Writer.WriteAsync(eventSource, cancellationToken);
        }
    }

    /// <summary>
    /// 从存储器中读取一条事件源
    /// </summary>
    /// <param name="cancellationToken">取消任务 Token</param>
    /// <returns>事件源对象</returns>
    public async ValueTask<IEventSource> ReadAsync(CancellationToken cancellationToken)
    {
        // 读取一条事件源
        var eventSource = await _channel.Reader.ReadAsync(cancellationToken);
        return eventSource;
    }

    /// <summary>
    /// 释放非托管资源
    /// </summary>
    public async void Dispose()
    {
        await _eventConsumer.Stop();
        GC.SuppressFinalize(this);
    }
}
```

## Redis 消息队列

Admin.NET 同时也集成了 Redis 消息队列静态工具类，直接使用即可。

```csharp
using NewLife.Caching.Queues;

namespace Admin.NET.Core;

/// <summary>
/// Redis 消息队列
/// </summary>
public static class RedisQueue
{
    private static readonly ICache _cache = App.GetRequiredService<ICache>();

    /// <summary>
    /// 获取普通队列
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="topic"></param>
    /// <returns></returns>
    public static IProducerConsumer<T> GetQueue<T>(string topic)
    {
        var queue = (_cache as FullRedis).GetQueue<T>(topic);
        return queue;
    }

    /// <summary>
    /// 发送一个数据到队列
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <returns></returns>
    public static int AddQueue<T>(string topic, T value)
    {
        var queue = GetQueue<T>(topic);
        return queue.Add(value);
    }

    /// <summary>
    /// 发送一个数据列表到队列
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static int AddQueueList<T>(string topic, List<T> value)
    {
        var queue = GetQueue<T>(topic);
        var count = queue.Count;
        var result = queue.Add(value.ToArray());
        return result - count;
    }

    /// <summary>
    /// 获取一批队列消息
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="topic"></param>
    /// <param name="count"></param>
    /// <returns></returns>
    public static List<T> Take<T>(string topic, int count = 1)
    {
        var queue = GetQueue<T>(topic);
        var result = queue.Take(count).ToList();
        return result;
    }

    /// <summary>
    /// 获取一个队列消息
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="topic"></param>
    /// <returns></returns>
    public static async Task<T> TakeOneAsync<T>(string topic)
    {
        var queue = GetQueue<T>(topic);
        return await queue.TakeOneAsync(1);
    }

    /// <summary>
    /// 获取可信队列，需要确认
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="topic"></param>
    /// <returns></returns>
    public static RedisReliableQueue<T> GetRedisReliableQueue<T>(string topic)
    {
        var queue = (_cache as FullRedis).GetReliableQueue<T>(topic);
        return queue;
    }

    /// <summary>
    /// 可信队列回滚
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="retryInterval"></param>
    /// <returns></returns>
    public static int RollbackAllAck(string topic, int retryInterval = 60)
    {
        var queue = GetRedisReliableQueue<string>(topic);
        queue.RetryInterval = retryInterval;
        return queue.RollbackAllAck();
    }

    /// <summary>
    /// 发送一个数据列表到可信队列
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static int AddReliableQueueList<T>(string topic, List<T> value)
    {
        var queue = (_cache as FullRedis).GetReliableQueue<T>(topic);
        var count = queue.Count;
        var result = queue.Add(value.ToArray());
        return result - count;
    }

    /// <summary>
    /// 发送一条数据到可信队列
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static int AddReliableQueue<T>(string topic, T value)
    {
        var queue = (_cache as FullRedis).GetReliableQueue<T>(topic);
        var count = queue.Count;
        var result = queue.Add(value);
        return result - count;
    }

    /// <summary>
    /// 在可信队列获取一条数据
    /// </summary>
    /// <param name="topic"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static T ReliableTakeOne<T>(string topic)
    {
        var queue = GetRedisReliableQueue<T>(topic);
        return queue.TakeOne(1);
    }

    /// <summary>
    /// 异步在可信队列获取一条数据
    /// </summary>
    /// <param name="topic"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static async Task<T> ReliableTakeOneAsync<T>(string topic)
    {
        var queue = GetRedisReliableQueue<T>(topic);
        return await queue.TakeOneAsync(1);
    }

    /// <summary>
    /// 在可信队列获取多条数据
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="count"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static List<T> ReliableTake<T>(string topic, int count)
    {
        var queue = GetRedisReliableQueue<T>(topic);
        return queue.Take(count).ToList();
    }

    /// <summary>
    /// 获取延迟队列
    /// </summary>
    /// <param name="topic"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static RedisDelayQueue<T> GetDelayQueue<T>(string topic)
    {
        var queue = (_cache as FullRedis).GetDelayQueue<T>(topic);
        return queue;
    }

    /// <summary>
    /// 发送一条数据到延迟队列
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <param name="delay">延迟时间。单位秒</param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static int AddDelayQueue<T>(string topic, T value, int delay)
    {
        var queue = GetDelayQueue<T>(topic);
        return queue.Add(value, delay);
    }

    /// <summary>
    /// 发送数据列表到延迟队列
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="value"></param>
    /// <param name="delay"></param>
    /// <typeparam name="T">延迟时间。单位秒</typeparam>
    /// <returns></returns>
    public static int AddDelayQueue<T>(string topic, List<T> value, int delay)
    {
        var queue = GetDelayQueue<T>(topic);
        queue.Delay = delay;
        return queue.Add(value.ToArray());
    }

    /// <summary>
    /// 异步在延迟队列获取一条数据
    /// </summary>
    /// <param name="topic"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static async Task<T> DelayTakeOne<T>(string topic)
    {
        var queue = GetDelayQueue<T>(topic);
        return await queue.TakeOneAsync(1);
    }

    /// <summary>
    /// 在延迟队列获取多条数据
    /// </summary>
    /// <param name="topic"></param>
    /// <param name="count"></param>
    /// <typeparam name="T"></typeparam>
    /// <returns></returns>
    public static List<T> DelayTake<T>(string topic, int count = 1)
    {
        var queue = GetDelayQueue<T>(topic);
        return queue.Take(count).ToList();
    }
}
```