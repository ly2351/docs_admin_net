---
sidebar_position: 4
---

# 4. 缓存管理

:::tip 提示

Admin.NET 支持内存缓存和 Redis 缓存两种模式，可以自由切换自行选择，配置文件在 `/Configuration/Cache.json`。
:::

```csharp
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

先注入缓存服务类, 然后进行缓存增删改查。至于当前缓存是内存缓存还是 Redis 模式完全由配置文件决定，应用操作层无感。

```csharp
/// <summary>
/// xxx服务
/// </summary>
[ApiDescriptionSettings(Order = 100)]
public class xxxService : IDynamicApiController, ITransient
{
    private readonly SysCacheService _sysCacheService;

    public xxxService(SysCacheService sysCacheService)
    {
        _sysCacheService = sysCacheService;
    }
}
```

该缓存服务类已经同时实现内存缓存和 Redis 缓存常用共同的接口。具体实现在 `/Admin.NET.Core/Service/Cache/SysCacheService.cs`

```csharp
/// <summary>
/// 系统缓存服务 🧩
/// </summary>
[ApiDescriptionSettings(Order = 400)]
public class SysCacheService : IDynamicApiController, ISingleton
{
    private readonly ICache _cache;
    private readonly CacheOptions _cacheOptions;

    public SysCacheService(ICache cache, IOptions<CacheOptions> cacheOptions)
    {
        _cache = cache;
        _cacheOptions = cacheOptions.Value;
    }

    /// <summary>
    /// 获取缓存键名集合 🔖
    /// </summary>
    /// <returns></returns>
    [DisplayName("获取缓存键名集合")]
    public List<string> GetKeyList()
    {
        return _cache == Cache.Default
            ? _cache.Keys.Where(u => u.StartsWith(_cacheOptions.Prefix)).Select(u => u[_cacheOptions.Prefix.Length..]).OrderBy(u => u).ToList()
            : ((FullRedis)_cache).Search($"{_cacheOptions.Prefix}*", int.MaxValue).Select(u => u[_cacheOptions.Prefix.Length..]).OrderBy(u => u).ToList();
    }

    /// <summary>
    /// 增加缓存
    /// </summary>
    /// <param name="key"></param>
    /// <param name="value"></param>
    /// <returns></returns>
    [NonAction]
    public bool Set(string key, object value)
    {
        if (string.IsNullOrWhiteSpace(key)) return false;
        return _cache.Set($"{_cacheOptions.Prefix}{key}", value);
    }

    /// <summary>
    /// 增加缓存并设置过期时间
    /// </summary>
    /// <param name="key"></param>
    /// <param name="value"></param>
    /// <param name="expire"></param>
    /// <returns></returns>
    [NonAction]
    public bool Set(string key, object value, TimeSpan expire)
    {
        if (string.IsNullOrWhiteSpace(key)) return false;
        return _cache.Set($"{_cacheOptions.Prefix}{key}", value, expire);
    }

    /// <summary>
    /// 获取缓存
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <returns></returns>
    [NonAction]
    public T Get<T>(string key)
    {
        return _cache.Get<T>($"{_cacheOptions.Prefix}{key}");
    }

    /// <summary>
    /// 删除缓存 🔖
    /// </summary>
    /// <param name="key"></param>
    /// <returns></returns>
    [ApiDescriptionSettings(Name = "Delete"), HttpPost]
    [DisplayName("删除缓存")]
    public int Remove(string key)
    {
        return _cache.Remove($"{_cacheOptions.Prefix}{key}");
    }

    /// <summary>
    /// 检查缓存是否存在
    /// </summary>
    /// <param name="key">键</param>
    /// <returns></returns>
    [NonAction]
    public bool ExistKey(string key)
    {
        return _cache.ContainsKey($"{_cacheOptions.Prefix}{key}");
    }

    /// <summary>
    /// 根据键名前缀删除缓存 🔖
    /// </summary>
    /// <param name="prefixKey">键名前缀</param>
    /// <returns></returns>
    [ApiDescriptionSettings(Name = "DeleteByPreKey"), HttpPost]
    [DisplayName("根据键名前缀删除缓存")]
    public int RemoveByPrefixKey(string prefixKey)
    {
        var delKeys = _cache == Cache.Default
            ? _cache.Keys.Where(u => u.StartsWith($"{_cacheOptions.Prefix}{prefixKey}")).ToArray()
            : ((FullRedis)_cache).Search($"{_cacheOptions.Prefix}{prefixKey}*", int.MaxValue).ToArray();
        return _cache.Remove(delKeys);
    }

    /// <summary>
    /// 根据键名前缀获取键名集合 🔖
    /// </summary>
    /// <param name="prefixKey">键名前缀</param>
    /// <returns></returns>
    [DisplayName("根据键名前缀获取键名集合")]
    public List<string> GetKeysByPrefixKey(string prefixKey)
    {
        return _cache == Cache.Default
            ? _cache.Keys.Where(u => u.StartsWith($"{_cacheOptions.Prefix}{prefixKey}")).Select(u => u[_cacheOptions.Prefix.Length..]).ToList()
            : ((FullRedis)_cache).Search($"{_cacheOptions.Prefix}{prefixKey}*", int.MaxValue).Select(u => u[_cacheOptions.Prefix.Length..]).ToList();
    }

    /// <summary>
    /// 获取缓存值 🔖
    /// </summary>
    /// <param name="key"></param>
    /// <returns></returns>
    [DisplayName("获取缓存值")]
    public object GetValue(string key)
    {
        return _cache == Cache.Default
            ? _cache.Get<object>($"{_cacheOptions.Prefix}{key}")
            : _cache.Get<string>($"{_cacheOptions.Prefix}{key}");
    }

    /// <summary>
    /// 获取或添加缓存（在数据不存在时执行委托请求数据）
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="callback"></param>
    /// <param name="expire">过期时间，单位秒</param>
    /// <returns></returns>
    [NonAction]
    public T GetOrAdd<T>(string key, Func<string, T> callback, int expire = -1)
    {
        if (string.IsNullOrWhiteSpace(key)) return default;
        return _cache.GetOrAdd($"{_cacheOptions.Prefix}{key}", callback, expire);
    }

    /// <summary>
    /// Hash匹配
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <returns></returns>
    [NonAction]
    public RedisHash<string, T> GetHashMap<T>(string key)
    {
        return _cache.GetDictionary<T>(key) as RedisHash<string, T>;
    }

    /// <summary>
    /// 批量添加HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="dic"></param>
    /// <returns></returns>
    [NonAction]
    public bool HashSet<T>(string key, Dictionary<string, T> dic)
    {
        var hash = GetHashMap<T>(key);
        return hash.HMSet(dic);
    }

    /// <summary>
    /// 添加一条HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="hashKey"></param>
    /// <param name="value"></param>
    [NonAction]
    public void HashAdd<T>(string key, string hashKey, T value)
    {
        var hash = GetHashMap<T>(key);
        hash.Add(hashKey, value);
    }

    /// <summary>
    /// 获取多条HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="fields"></param>
    /// <returns></returns>
    [NonAction]
    public List<T> HashGet<T>(string key, params string[] fields)
    {
        var hash = GetHashMap<T>(key);
        var result = hash.HMGet(fields);
        return result.ToList();
    }

    /// <summary>
    /// 获取一条HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="field"></param>
    /// <returns></returns>
    [NonAction]
    public T HashGetOne<T>(string key, string field)
    {
        var hash = GetHashMap<T>(key);
        var result = hash.HMGet(new string[] { field });
        return result[0];
    }

    /// <summary>
    /// 根据KEY获取所有HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <returns></returns>
    [NonAction]
    public IDictionary<string, T> HashGetAll<T>(string key)
    {
        var hash = GetHashMap<T>(key);
        return hash.GetAll();
    }

    /// <summary>
    /// 删除HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="fields"></param>
    /// <returns></returns>
    [NonAction]
    public int HashDel<T>(string key, params string[] fields)
    {
        var hash = GetHashMap<T>(key);
        return hash.HDel(fields);
    }

    /// <summary>
    /// 搜索HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="searchModel"></param>
    /// <returns></returns>
    [NonAction]
    public List<KeyValuePair<string, T>> HashSearch<T>(string key, SearchModel searchModel)
    {
        var hash = GetHashMap<T>(key);
        return hash.Search(searchModel).ToList();
    }

    /// <summary>
    /// 搜索HASH
    /// </summary>
    /// <typeparam name="T"></typeparam>
    /// <param name="key"></param>
    /// <param name="pattern"></param>
    /// <param name="count"></param>
    /// <returns></returns>
    [NonAction]
    public List<KeyValuePair<string, T>> HashSearch<T>(string key, string pattern, int count)
    {
        var hash = GetHashMap<T>(key);
        return hash.Search(pattern, count).ToList();
    }
}
```