---
sidebar_position: 25
---

# 25. 远程请求

:::tip 提示

Admin.NET 提供两种方式访问发送远程请求，`IHttpDispatchProxy` 代理方式 和 `字符串拓展`方式。**禁止非法爬虫**。
:::

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
<TabItem value="代理模式字符串模式" label="代理模式字符串模式">

```csharp
public interface IHttp : IHttpDispatchProxy
{
    // 发送 Get 请求
    [Get("http://admin.net/get")]
    Task<xxx> GetXXXAsync();

    // 发送 Post 请求
    [Post("http://admin.net/post")]
    Task<xxx> PostXXXAsync();

    // 其他请求方式类同
}
```

</TabItem>
<TabItem value="字符串模式" label="字符串模式">

```csharp
var response = await "http://furion.baiqian.ltd/get".GetAsync();

var response = await "http://furion.baiqian.ltd/post".PostAsync();

var response = await "http://furion.baiqian.ltd/put".PutAsync();

var response = await "http://furion.baiqian.ltd/delete".DeleteAsync();

var response = await "http://furion.baiqian.ltd/patch".PatchAsync();

var response = await "http://furion.baiqian.ltd/head".HeadAsync();
```

</TabItem>
</Tabs>

也推荐学习下 微软自带 `HttpClient` 类教程 使用 [HttpClient 类发出 HTTP 请求](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/networking/http/httpclient?redirectedfrom=MSDN)

:::info 相关信息

同时，也推荐使用 Flurl 组件。 Flurl 是一个现代的、流畅的、异步的、可测试的、可移植的、充满流行语的 URL 构建器和 HTTP 客户端库，非常方便强悍。

开源仓库 [https://github.com/tmenier/Flurl](https://github.com/tmenier/Flurl)。

使用教程 [https://flurl.dev/docs/fluent-http/](https://flurl.dev/docs/fluent-http/)
:::

远程请求-字符串方式使用举例：`IHttpDispatchProxy 代理方式使用与此类似`

1、请求方式

```csharp
await "http://admin.net/post".SetHttpMethod(HttpMethod.Get);
```
2、地址模板（区分大小写）

```csharp
await "http://admin.net/post/{id}?name={name}".SetTemplates(new {
    id = 1,
    name = "Admin.NET"
});
```

3、请求报文头

```csharp
await "http://admin.net/post".SetHeaders(new {
    Authorization = "Bearer 你的token"
});
```

4、URL 地址参数

```csharp
await "http://admin.net/get".SetQueries(new {
    id = 1,
    name = "Admin.NET"
});
```

5、URL 请求客户端

```csharp
await "http://admin.net".SetClient(() => new HttpClient());
```

6、Body 参数

```csharp
await "http://admin.net/api/user/add".SetBody(new { Id = 1, Name = "Admin.NET" }, "application/json");
```

:::warning 注意

若请求 Content-Type 设置为 application/x-www-form-urlencoded 类型，那么框架底层自动将数据进行 UrlEncode 编码处理，无需外部处理。
:::

7、Content-Type

```csharp
await "http://admin.net/post".SetContentType("application/json");
```

8、内容编码

```csharp
await "http://admin.net/post".SetContentEncoding(Encoding.UTF8);
```

9、JSON 序列化

```csharp
await "http://admin.net/post".SetJsonSerialization<NewtonsoftJsonSerializerProvider>(new JsonSerializerSettings {

});
```

10、Body 参数验证

```csharp
await "http://admin.net/post".SetValidationState(includeNull: true); // 支持类中 [Required] 等完整模型验证特性
```

11、请求拦截

```csharp
await "http://admin.net/".OnRequesting((client, req) => {
    req.AppendQueries(new Dictionary<string, object> {
        { "token", "xxxx"}
    });
});
```

12、HttpClient 拦截

```csharp
await "http://admin.net/".OnClientCreating(client => {
    client.Timeout = TimeSpan.FromSeconds(30); // 超时
});
```

13、请求之前拦截

```csharp
await "http://admin.net/".OnRequesting((client, req) => {

});
```

14、请求结果拦截

```csharp
await "http://admin.net/".OnResponsing((client, req) => {

});
```

15、请求异常拦截

```csharp
await "http://admin.net/".OnException((client, res, errors) => {
  
});
```

16、文件上传

```csharp
// 支持多个文件、支持bytes、fileStream格式，bytes 可通过 File.ReadAllBytes (文件路径) 获取
var res = await "http://admin.net//upload".SetContentType("multipart/form-data")
    .SetFiles(("key", bytes, "fileName")).PostAsync();
    ```

:::note相关信息

若遇到微信上传出现问题，则可设置 Content-Type 为：application/octet-stream。
:::

17、重试策略

```csharp
var res = await "http://admin.net".SetRetryPolicy(5, 1000).GetAsync(); // 若请求失败则则重试 5 次，每次间隔 1s
```

18、GZip 压缩

```csharp
var res = await "http://admin.net".WithGZip().GetAsync();
```

19、Url 转码

```csharp
var res = await "http://admin.net".WithEncodeUrl(false).GetAsync();
```

20、HTTP 版本

```csharp
var res = await "http://admin.net".SetHttpVersion("2.0").GetAsync(); // 处理 HTTP 和 HTTPS 请求异常
```

21、下载文件

```csharp
await "http://admin.net/logo.png".GetToSaveAsync("D:/1.png");
```

22、设置 Cookies

```csharp
await "http://admin.net/post"
    .OnRequesting((client, request) =>
    {
        var cookieContainer = new CookieContainer();
        cookieContainer.Add(request.RequestUri, new Cookie("key", "Admin.NET"));
        request.Headers.Add("Cookie", cookieContainer.GetCookieHeader(request.RequestUri));
    })
    .GetAsync();
```

23、静态方式

```csharp
await HttpRequestPart.Default().SetRequestUrl("http://admin.net").GetAsStringAsync();
```

24、获取 Cookies

```csharp
using var response = await "http://admin.net".GetAsync();
var cookies = response.GetCookies();
```