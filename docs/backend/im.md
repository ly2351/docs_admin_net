---
sidebar_position: 10
---

# 10. 即时通讯


:::tip 提示

Admin.NET 即时通讯采用微软组件 `ASP.NET Core SignalR`，前端采用组件NPM包 `@microsoft/signalr`，后面考虑换成组件 MQTT。
:::

目前系统采用 即时通讯 功能有 【在线用户】相关处理模块。实时统计显示在线用户列表及数量，可以踢某人下线等。具体实现类 Admin.NET.Core/Hub/OnlineUserHub.cs，后台代码如下：

```csharp
using Furion.InstantMessaging;
using Microsoft.AspNetCore.SignalR;

namespace Admin.NET.Core;

/// <summary>
/// 在线用户集线器
/// </summary>
[MapHub("/hubs/onlineUser")]
public class OnlineUserHub : Hub<IOnlineUserHub>
{
    private const string GROUP_ONLINE = "GROUP_ONLINE_"; // 租户分组前缀

    private readonly SqlSugarRepository<SysOnlineUser> _sysOnlineUerRep;
    private readonly SysMessageService _sysMessageService;
    private readonly IHubContext<OnlineUserHub, IOnlineUserHub> _onlineUserHubContext;
    private readonly SysCacheService _sysCacheService;
    private readonly SysConfigService _sysConfigService;

    public OnlineUserHub(SqlSugarRepository<SysOnlineUser> sysOnlineUerRep,
        SysMessageService sysMessageService,
        IHubContext<OnlineUserHub, IOnlineUserHub> onlineUserHubContext,
        SysCacheService sysCacheService,
        SysConfigService sysConfigService)
    {
        _sysOnlineUerRep = sysOnlineUerRep;
        _sysMessageService = sysMessageService;
        _onlineUserHubContext = onlineUserHubContext;
        _sysCacheService = sysCacheService;
        _sysConfigService = sysConfigService;
    }

    /// <summary>
    /// 连接
    /// </summary>
    /// <returns></returns>
    public override async Task OnConnectedAsync()
    {
        var httpContext = Context.GetHttpContext();
        var token = httpContext.Request.Query["access_token"];
        var claims = JWTEncryption.ReadJwtToken(token)?.Claims;
        var client = Parser.GetDefault().Parse(httpContext.Request.Headers["User-Agent"]);

        var userId = claims?.FirstOrDefault(u => u.Type == ClaimConst.UserId)?.Value;
        var tenantId = claims?.FirstOrDefault(u => u.Type == ClaimConst.TenantId)?.Value;
        var user = new SysOnlineUser
        {
            ConnectionId = Context.ConnectionId,
            UserId = string.IsNullOrWhiteSpace(userId) ? 0 : long.Parse(userId),
            UserName = claims?.FirstOrDefault(u => u.Type == ClaimConst.Account)?.Value,
            RealName = claims?.FirstOrDefault(u => u.Type == ClaimConst.RealName)?.Value,
            Time = DateTime.Now,
            Ip = httpContext.Connection.RemoteIpAddress.MapToIPv4().ToString(),
            Browser = client.UA.Family + client.UA.Major,
            Os = client.OS.Family + client.OS.Major,
            TenantId = string.IsNullOrWhiteSpace(tenantId) ? 0 : Convert.ToInt64(tenantId),
        };
        await _sysOnlineUerRep.InsertAsync(user);

        // 是否开启单用户登录
        if (await _sysConfigService.GetConfigValue<bool>(CommonConst.SysSingleLogin))
        {
            _sysCacheService.Set(CacheConst.KeyUserOnline + user.UserId, user);
        }
        else
        {
            var device = (client.UA.Family + client.UA.Major + client.OS.Family + client.OS.Major).Trim();
            _sysCacheService.Set(CacheConst.KeyUserOnline + user.UserId + device, user);
        }

        // 以租户Id进行分组
        var groupName = $"{GROUP_ONLINE}{user.TenantId}";
        await _onlineUserHubContext.Groups.AddToGroupAsync(Context.ConnectionId, groupName);

        var userList = await _sysOnlineUerRep.AsQueryable().Filter("", true)
            .Where(u => u.TenantId == user.TenantId).Take(10).ToListAsync();
        await _onlineUserHubContext.Clients.Groups(groupName).OnlineUserList(new OnlineUserList
        {
            RealName = user.RealName,
            Online = true,
            UserList = userList
        });
    }

    /// <summary>
    /// 断开
    /// </summary>
    /// <param name="exception"></param>
    /// <returns></returns>
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        if (string.IsNullOrEmpty(Context.ConnectionId)) return;

        var httpContext = Context.GetHttpContext();
        var client = Parser.GetDefault().Parse(httpContext.Request.Headers["User-Agent"]);

        var user = await _sysOnlineUerRep.AsQueryable().Filter("", true).FirstAsync(u => u.ConnectionId == Context.ConnectionId);
        if (user == null) return;

        await _sysOnlineUerRep.DeleteAsync(u => u.Id == user.Id);

        // 是否开启单用户登录
        if (await _sysConfigService.GetConfigValue<bool>(CommonConst.SysSingleLogin))
        {
            _sysCacheService.Remove(CacheConst.KeyUserOnline + user.UserId);
        }
        else
        {
            var device = (client.UA.Family + client.UA.Major + client.OS.Family + client.OS.Major).Trim();
            _sysCacheService.Remove(CacheConst.KeyUserOnline + user.UserId + device);
        }

        // 通知当前组用户变动
        var userList = await _sysOnlineUerRep.AsQueryable().Filter("", true)
            .Where(u => u.TenantId == user.TenantId).Take(10).ToListAsync();
        await _onlineUserHubContext.Clients.Groups($"{GROUP_ONLINE}{user.TenantId}").OnlineUserList(new OnlineUserList
        {
            RealName = user.RealName,
            Online = false,
            UserList = userList
        });
    }

    /// <summary>
    /// 强制下线
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    public async Task ForceOffline(OnlineUserHubInput input)
    {
        await _onlineUserHubContext.Clients.Client(input.ConnectionId).ForceOffline("强制下线");
    }

    /// <summary>
    /// 发送信息给某个人
    /// </summary>
    /// <param name="message"></param>
    /// <returns></returns>
    public async Task ClientsSendMessage(MessageInput message)
    {
        await _sysMessageService.SendUser(message);
    }

    /// <summary>
    /// 发送信息给所有人
    /// </summary>
    /// <param name="message"></param>
    /// <returns></returns>
    public async Task ClientsSendMessagetoAll(MessageInput message)
    {
        await _sysMessageService.SendAllUser(message);
    }

    /// <summary>
    /// 发送消息给某些人（除了本人）
    /// </summary>
    /// <param name="message"></param>
    /// <returns></returns>
    public async Task ClientsSendMessagetoOther(MessageInput message)
    {
        await _sysMessageService.SendOtherUser(message);
    }

    /// <summary>
    /// 发送消息给某些人
    /// </summary>
    /// <param name="message"></param>
    /// <returns></returns>
    public async Task ClientsSendMessagetoUsers(MessageInput message)
    {
        await _sysMessageService.SendUsers(message);
    }
}
```

后台可以直接注入此服务类进行即时消息处理，比如发送消息 ClientsSendMessage 方法，根据不同参数调用即可。

前端实现具体实现与调用文件路径 src/views/system/onlineUser/signalR.ts

```csharp
import * as SignalR from '@microsoft/signalr';
import { ElNotification } from 'element-plus';
import { getToken } from '/@/utils/axios-utils';

// 初始化SignalR对象
const connection = new SignalR.HubConnectionBuilder()
 .configureLogging(SignalR.LogLevel.Information)
 .withUrl(`${window.__env__.VITE_API_URL}/hubs/onlineUser?access_token=${getToken()}`, { transport: SignalR.HttpTransportType.WebSockets, skipNegotiation: true })
 .withAutomaticReconnect({
  nextRetryDelayInMilliseconds: () => {
   return 5000; // 每5秒重连一次
  },
 })
 .build();

connection.keepAliveIntervalInMilliseconds = 15 * 1000; // 心跳检测15s
connection.serverTimeoutInMilliseconds = 30 * 60 * 1000; // 超时时间30m

// 启动连接
connection.start().then(() => {
 console.log('启动连接');
});
// 断开连接
connection.onclose(async () => {
 console.log('断开连接');
});
// 重连中
connection.onreconnecting(() => {
 ElNotification({
  title: '提示',
  message: '服务器已断线...',
  type: 'error',
  position: 'bottom-right',
 });
});
// 重连成功
connection.onreconnected(() => {
 console.log('重连成功');
});

connection.on('OnlineUserList', () => {});

export { connection as signalR };
```

前端使用的时候导入 `import { signalR } from './signalR';`，然后调用方法即可 signalR.on('xxx函数名') 。

```csharp
 signalR.off('OnlineUserList');

 signalR.on('OnlineUserList', (data: any) => {
 
  };
 });
```

**用户区分** 注入 `UserIdProvider`，调用 `GetUserId` 方法即可获取当前连接用户Id。

```csharp
public interface UserIdProvider : IUserIdProvider
{
    public new string GetUserId(HubConnectionContext connection)
    {
        return connection.User?.Claims?.FirstOrDefault(u => u.Type == ClaimConst.UserId)?.Value;
    }
}
```