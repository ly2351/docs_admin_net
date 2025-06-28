---
sidebar_position: 21
---

# 21. 令牌Token

:::tip 提示

Admin.NET 采用 Token 鉴权，并且是双令牌模式。访问 AccessToken 和刷新 RefreshToken，AccessToken 过期后，系统会根据刷新 RefreshToken 去服务器换新的 AccessToken，前端更新保存 Token 可以继续访问接口。

如果需要对特定的 Action 或 Controller 允许匿名访问，则只需要贴 [AllowAnonymous] 即可。

同时，Admin.NET 已经实现了当前用户管理服务类，直接注入则可以直接获取当前登录用户的名称、用户Id等信息，无需再根据 Token 手动解析。直接拿到当前访问用户Id，可方便处理具体业务逻辑。

:::

JWT 配置文件如下 Configuration/JWT.json，记得更换密钥串，防止多个系统一样，造成账号安全事故。密匙串建议直接用32位长度的MD5串。.NET8 时所需要的字符串长度更长，就再复制一份。

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "JWTSettings": {
    "ValidateIssuerSigningKey": true, // 是否验证密钥，bool 类型，默认true
    "IssuerSigningKey": "3c1cbc3f546eda35168c3aa3cb91780fbe703f0996c6d123ea96dc85c70bbc0a", // 密钥，string 类型，必须是复杂密钥，长度大于16
    "ValidateIssuer": true, // 是否验证签发方，bool 类型，默认true
    "ValidIssuer": "Admin.NET", // 签发方，string 类型
    "ValidateAudience": true, // 是否验证签收方，bool 类型，默认true
    "ValidAudience": "Admin.NET", // 签收方，string 类型
    "ValidateLifetime": true, // 是否验证过期时间，bool 类型，默认true，建议true
    //"ExpiredTime": 20, // 过期时间，long 类型，单位分钟，默认20分钟，最大支持 13 年
    "ClockSkew": 5, // 过期时间容错值，long 类型，单位秒，默认5秒
    "Algorithm": "HS256", // 加密算法，string 类型，默认 HS256
    "RequireExpirationTime": true // 验证过期时间，设置 false 将永不过期
  }
}
```

:::info Algorithm 加密算法

- HS256
- HS384
- HS512
- PS256
- PS384
- PS512
- ES256
- ES256K
- ES384
- ES512
- EdDSA

具体可参考 [SecurityAlgorithms](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/blob/dev/src/Microsoft.IdentityModel.Tokens/SecurityAlgorithms.cs)

Token 默认过期时间为 7 天，在系统参数配置里面可自行修改。

当前登录用户管理服务类 [Admin.NET.Core/Service/User/UserManager.cs](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Admin.NET/Admin.NET.Core/Service/User/UserManager.cs) 实现如下：

```csharp
namespace Admin.NET.Core;

/// <summary>
/// 当前登录用户
/// </summary>
public class UserManager : IScoped
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    /// <summary>
    /// 用户ID
    /// </summary>
    public long UserId => (_httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.UserId)?.Value).ToLong();

    /// <summary>
    /// 租户ID
    /// </summary>
    public long TenantId => (_httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.TenantId)?.Value).ToLong();

    /// <summary>
    /// 用户账号
    /// </summary>
    public string Account => _httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.Account)?.Value;

    /// <summary>
    /// 真实姓名
    /// </summary>
    public string RealName => _httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.RealName)?.Value;

    /// <summary>
    /// 是否超级管理员
    /// </summary>
    public bool SuperAdmin => _httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.AccountType)?.Value == ((int)AccountTypeEnum.SuperAdmin).ToString();

    /// <summary>
    /// 是否系统管理员
    /// </summary>
    public bool SysAdmin => _httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.AccountType)?.Value == ((int)AccountTypeEnum.SysAdmin).ToString();

    /// <summary>
    /// 组织机构Id
    /// </summary>
    public long OrgId => (_httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.OrgId)?.Value).ToLong();

    /// <summary>
    /// 微信OpenId
    /// </summary>
    public string OpenId => _httpContextAccessor.HttpContext?.User.FindFirst(ClaimConst.OpenId)?.Value;

    public UserManager(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
}
```


:::tip 提示

若默认的 Claim 里面内容不满足业务无需，可重新登录接口自行设定 Claim 内容。**系统内置服务接口一般都是 partial 类型，并且都是虚方法 virtual，实体类扩展也类似。** 比如移动端登录时自定义 Token时，根据业务需要塞进 Claim 里面内容所需关键字即可，此时记得扩展用户管理类 [Admin.NET.Core/Service/User/UserManager.cs](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Admin.NET/Admin.NET.Core/Service/User/UserManager.cs)，继承此类即可。
:::

创建 Token 示例：

```csharp
    /// <summary>
    /// 生成Token令牌 🔖
    /// </summary>
    /// <param name="user"></param>
    /// <returns></returns>
    [NonAction]
    public virtual async Task<LoginOutput> CreateToken(SysUser user)
    {
        // 单用户登录
        await _sysOnlineUserService.SingleLogin(user.Id);

        // 生成Token令牌
        var tokenExpire = await _sysConfigService.GetTokenExpire();
        var accessToken = JWTEncryption.Encrypt(new Dictionary<string, object>
        {
            { ClaimConst.UserId, user.Id },
            { ClaimConst.TenantId, user.TenantId },
            { ClaimConst.Account, user.Account },
            { ClaimConst.RealName, user.RealName },
            { ClaimConst.AccountType, user.AccountType },
            { ClaimConst.OrgId, user.OrgId },
            { ClaimConst.OrgName, user.SysOrg?.Name },
            { ClaimConst.OrgType, user.SysOrg?.Type },
        }, tokenExpire);

        // 生成刷新Token令牌
        var refreshTokenExpire = await _sysConfigService.GetRefreshTokenExpire();
        var refreshToken = JWTEncryption.GenerateRefreshToken(accessToken, refreshTokenExpire);

        // 设置响应报文头
        _httpContextAccessor.HttpContext.SetTokensOfResponseHeaders(accessToken, refreshToken);

        // Swagger Knife4UI-AfterScript登录脚本
        // ke.global.setAllHeader('Authorization', 'Bearer ' + ke.response.headers['access-token']);

        return new LoginOutput
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken
        };
    }
```

**后端授权** 实现，自动验证、刷新 [Token Admin.NET.Web.Core/Handlers/JwtHandler.cs](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Admin.NET/Admin.NET.Web.Core/Handlers/JwtHandler.cs)

```csharp
using Admin.NET.Core;
using Admin.NET.Core.Service;
using Furion;
using Furion.Authorization;
using Furion.DataEncryption;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Threading.Tasks;

namespace Admin.NET.Web.Core
{
    public class JwtHandler : AppAuthorizeHandler
    {
        private readonly IServiceProvider _serviceProvider;

        public JwtHandler(IServiceProvider serviceProvider)
        {
            _serviceProvider = serviceProvider;
        }

        /// <summary>
        /// 自动刷新Token
        /// </summary>
        /// <param name="context"></param>
        /// <param name="httpContext"></param>
        /// <returns></returns>
        public override async Task HandleAsync(AuthorizationHandlerContext context, DefaultHttpContext httpContext)
        {
            // var serviceProvider = context.GetCurrentHttpContext().RequestServices;
            using var serviceScope = _serviceProvider.CreateScope();

            // 若当前账号存在黑名单中则授权失败
            var sysCacheService = serviceScope.ServiceProvider.GetRequiredService<SysCacheService>();
            if (sysCacheService.ExistKey($"{CacheConst.KeyBlacklist}{context.User.FindFirst(ClaimConst.UserId)?.Value}"))
            {
                context.Fail();
                context.GetCurrentHttpContext().SignoutToSwagger();
                return;
            }

            var sysConfigService = serviceScope.ServiceProvider.GetRequiredService<SysConfigService>();
            var tokenExpire = await sysConfigService.GetTokenExpire();
            var refreshTokenExpire = await sysConfigService.GetRefreshTokenExpire();
            if (JWTEncryption.AutoRefreshToken(context, context.GetCurrentHttpContext(), tokenExpire, refreshTokenExpire))
            {
                await AuthorizeHandleAsync(context);
            }
            else
            {
                context.Fail(); // 授权失败
                var currentHttpContext = context.GetCurrentHttpContext();
                if (currentHttpContext == null)
                    return;
                // 跳过由于 SignatureAuthentication 引发的失败
                if (currentHttpContext.Items.ContainsKey(SignatureAuthenticationDefaults.AuthenticateFailMsgKey))
                    return;
                currentHttpContext.SignoutToSwagger();
            }
        }

        public override async Task<bool> PipelineAsync(AuthorizationHandlerContext context, DefaultHttpContext httpContext)
        {
            // 已自动验证 Jwt Token 有效性
            return await CheckAuthorizeAsync(httpContext);
        }

        /// <summary>
        /// 权限校验核心逻辑
        /// </summary>
        /// <param name="httpContext"></param>
        /// <returns></returns>
        private static async Task<bool> CheckAuthorizeAsync(DefaultHttpContext httpContext)
        {
            // 排除超管权限
            if (App.User.FindFirst(ClaimConst.AccountType)?.Value == ((int)AccountTypeEnum.SuperAdmin).ToString())
                return true;

            // 接口路由权限
            var path = httpContext.Request.Path.ToString();
            var apis = await App.GetRequiredService<SysRoleService>().GetUserApiList();
            return apis.Exists(u => path.Contains(u, StringComparison.CurrentCulture));
        }
    }
}
```

前端通过 [Web/src/utils/axios-utils.ts](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Web/src/utils/axios-utils.ts) 自动解析 Token，访问自动带上 Token，自动判断是否过期、用刷新 Token 去换新 Token、更新本地 Token 都是自动化，无需手动处理。

```typescript
/**
 * 解密 JWT token 的信息
 * @param token jwt token 字符串
 * @returns <any>object
 */
export function decryptJWT(token: string): any {
 token = token.replace(/_/g, '/').replace(/-/g, '+');
 var json = decodeURIComponent(escape(window.atob(token.split('.')[1])));
 return JSON.parse(json);
}
```