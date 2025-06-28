---
sidebar_position: 12
---

# 12. 接口限流


:::tip 提示

`AspNetCoreRateLimit` 是一个 ASP.NET Core 速率限制的解决方案，旨在控制客户端根据IP地址或客户端ID向 WebAPI 或 MVC 应用发出的请求的速率。包含一个 `IpRateLimitMiddleware` 和`ClientRateLimitMiddleware`，每个中间件可以根据不同的场景配置限制允许IP或客户端，自定义这些限制策略，也可以将限制策略应用在每​​个API URL或具体的HTTP Method上。

`AspNetCoreRateLimit` 支持了两种方式：基于客户端IP（IpRateLimitMiddleware）和客户端ID（ClientRateLimitMiddleware）速率限制。
:::

直接修改接口限流配置文件就可以了 /Configuration/Limit.json，系统默认启用接口限流:

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  // IP限流配置
  "IpRateLimiting": {
    // 例如:设置每分钟5次访问限流
    // 当False时：每个接口都加入计数，不管你访问哪个接口，只要在一分钟内累计够5次，将禁止访问。
    // 当True 时：当一分钟请求了5次GetData接口，则该接口将在时间段内禁止访问，但是还可以访问PostData()5次,总得来说是每个接口都有5次在这一分钟，互不干扰。
    "EnableEndpointRateLimiting": true,
    // 如果StackBlockedRequests设置为false，拒绝的API调用不会添加到调用次数计数器上。比如：如果客户端每秒发出3个请求并且您设置了每秒一个调用的限制，
    // 则每分钟或每天计数器等其他限制将仅记录第一个调用，即成功的API调用。如果您希望被拒绝的API调用计入其他时间的显示（分钟，小时等），则必须设置
    "StackBlockedRequests": false,
    // 在RealIpHeader使用时，你的Kestrel服务器背后是一个反向代理，如果你的代理服务器使用不同的页眉然后提取客户端IP X-Real-IP使用此选项来设置它。
    "RealIpHeader": "X-Real-IP",
    // 将ClientIdHeader被用于提取白名单的客户端ID。如果此标头中存在客户端ID并且与ClientWhitelist中指定的值匹配，则不应用速率限制。
    "ClientIdHeader": "X-ClientId",
    // IP白名单:支持Ipv4和Ipv6
    "IpWhitelist": [],
    // 端点白名单
    "EndpointWhitelist": [],
    // 客户端白名单
    "ClientWhitelist": [],
    "QuotaExceededResponse": {
      "Content": "{{\"code\":429,\"type\":\"error\",\"message\":\"访问过于频繁,请稍后重试!\",\"result\":null,\"extras\":null}}",
      "ContentType": "application/json",
      "StatusCode": 429
    },
    // 返回状态码
    "HttpStatusCode": 429,
    // API规则,结尾一定要带*
    "GeneralRules": [
      // 1秒钟只能调用10次
      {
        "Endpoint": "*",
        "Period": "1s",
        "Limit": 10
      },
      // 1分钟只能调用600次
      {
        "Endpoint": "*",
        "Period": "1m",
        "Limit": 600
      },
      // 1小时只能调用3600
      {
        "Endpoint": "*",
        "Period": "1h",
        "Limit": 3600
      },
      // 1天只能调用86400次
      {
        "Endpoint": "*",
        "Period": "1d",
        "Limit": 86400
      }
    ]
  },
  "IpRateLimitPolicies": {
    "IpRules": [
      {
        "Ip": "XXX.XXX.XXX.XXX",
        "Rules": [
          {
            "Endpoint": "*",
            "Period": "1s",
            "Limit": 10
          },
          {
            "Endpoint": "*",
            "Period": "1m",
            "Limit": 600
          }
        ]
      }
    ]
  },
  // 客户端限流配置
  "ClientRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "ClientIdHeader": "X-ClientId",
    "EndpointWhitelist": [],
    "ClientWhitelist": [],
    "QuotaExceededResponse": {
      "Content": "{{\"code\":429,\"type\":\"error\",\"message\":\"访问人数过多,请稍后重试!\",\"result\":null,\"extras\":null}}",
      "ContentType": "application/json",
      "StatusCode": 429
    },
    "HttpStatusCode": 429,
    "GeneralRules": [
      {
        "Endpoint": "*",
        "Period": "1s",
        "Limit": 10
      },
      {
        "Endpoint": "*",
        "Period": "1m",
        "Limit": 600
      }
    ]
  },
  "ClientRateLimitPolicies": {
    "ClientRules": [
      {
        "ClientId": "xxx-xxx",
        "Rules": [
          {
            "Endpoint": "*",
            "Period": "1s",
            "Limit": 10
          },
          {
            "Endpoint": "*",
            "Period": "1m",
            "Limit": 600
          }
        ]
      }
    ]
  }
}
```

:::info 相关信息

.NET8 已经内置限流组件，Admin.NET 将会去掉 AspNetCoreRateLimit 限流组件。ASP.NET Core 中的速率限制中间件
:::

以下采用了 Fixed Window （固定窗口）的策略来进行限流，四种常用的限流策略（固定/滑动窗口、令牌桶、并发）。

```csharp
services.AddRateLimiter(options => {
  options.AddFixedWindowLimiter(RateLimitPolicies.Fixed, opt => {
    opt.Window = TimeSpan.FromMinutes(1); // 时间窗口
    opt.PermitLimit = 3; // 在时间窗口内允许的最大请求数
    opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst; // 请求处理顺序
    opt.QueueLimit = 2; // 队列中允许的最大请求数
  });
});
```
