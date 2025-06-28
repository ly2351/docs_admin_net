---
sidebar_position: 11
---

# 11. 统一认证


:::tip 提示

Admin.NET 第三方授权登录采用组件 `AspNet.Security.OAuth.Providers`。系统已经继承 `AspNet.Security.OAuth.Gitee`、`AspNet.Security.OAuth.Weixin`，其他三方可自行添加。具体使用流程可参考教程 GitHub第三方授权登录
:::

配置文件 OAuth.json，每个平台的 `ClientId`、`ClientSecret`请自行填写，并根据部署地址，填写指定的登录回调地址。
```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "OAuth": {
    "Weixin": {
      "ClientId": "xxx",
      "ClientSecret": "xxx"
    },
    "Gitee": {
      "ClientId": "xxx",
      "ClientSecret": "xxx"
    }
  }
}
```

统一授权第三方登录的时候，在登录回调地址里面处理登录权限逻辑。系统会自动创建账号及默认分配角色权限，系统表 `SysOAuthUser` 存储三方账号相关，系统账号表 `SysUser` 则保存其对应系统账号信息。若已存在则直接进行系统登录。具体实现类见 `SysOAuthService.cs` ，此时前端登录会根据 `/login?token={token.AccessToken}` 进行路由参数解析 Token。

```csharp
using Microsoft.AspNetCore.Authentication;
using System.Security.Claims;

namespace Admin.NET.Core.Service;

/// <summary>
/// 系统OAuth服务 🧩
/// </summary>
[AllowAnonymous]
[ApiDescriptionSettings(Order = 498)]
public class SysOAuthService : IDynamicApiController, ITransient
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly SqlSugarRepository<SysOAuthUser> _sysOAuthUserRep;

    public SysOAuthService(IHttpContextAccessor httpContextAccessor,
        SqlSugarRepository<SysOAuthUser> sysOAuthUserRep)
    {
        _httpContextAccessor = httpContextAccessor;
        _sysOAuthUserRep = sysOAuthUserRep;
    }

    /// <summary>
    /// 第三方登录 🔖
    /// </summary>
    /// <param name="provider"></param>
    /// <param name="redirectUrl"></param>
    /// <returns></returns>
    [ApiDescriptionSettings(Name = "SignIn"), HttpGet]
    [DisplayName("第三方登录")]
    public virtual async Task<IActionResult> SignIn([FromQuery] string provider, [FromQuery] string redirectUrl)
    {
        if (string.IsNullOrWhiteSpace(provider) || !await _httpContextAccessor.HttpContext.IsProviderSupportedAsync(provider))
            throw Oops.Oh("不支持的OAuth类型");

        var request = _httpContextAccessor.HttpContext.Request;
        var url = $"{request.Scheme}://{request.Host}{request.PathBase}{request.Path}Callback?provider={provider}&redirectUrl={redirectUrl}";
        var properties = new AuthenticationProperties { RedirectUri = url };
        properties.Items["LoginProvider"] = provider;
        return await Task.FromResult(new ChallengeResult(provider, properties));
    }

    /// <summary>
    /// 授权回调 🔖
    /// </summary>
    /// <param name="provider"></param>
    /// <param name="redirectUrl"></param>
    /// <returns></returns>
    [ApiDescriptionSettings(Name = "SignInCallback"), HttpGet]
    [DisplayName("授权回调")]
    public virtual async Task<IActionResult> SignInCallback([FromQuery] string provider = null, [FromQuery] string redirectUrl = "")
    {
        if (string.IsNullOrWhiteSpace(provider) || !await _httpContextAccessor.HttpContext.IsProviderSupportedAsync(provider))
            throw Oops.Oh("不支持的OAuth类型");

        var authenticateResult = await _httpContextAccessor.HttpContext.AuthenticateAsync(provider);
        if (!authenticateResult.Succeeded)
            throw Oops.Oh("授权失败");

        var openIdClaim = authenticateResult.Principal.FindFirst(ClaimTypes.NameIdentifier);
        if (openIdClaim == null || string.IsNullOrWhiteSpace(openIdClaim.Value))
            throw Oops.Oh("授权失败");

        var name = authenticateResult.Principal.FindFirst(ClaimTypes.Name)?.Value;
        var email = authenticateResult.Principal.FindFirst(ClaimTypes.Email)?.Value;
        var mobilePhone = authenticateResult.Principal.FindFirst(ClaimTypes.MobilePhone)?.Value;
        var dateOfBirth = authenticateResult.Principal.FindFirst(ClaimTypes.DateOfBirth)?.Value;
        var gender = authenticateResult.Principal.FindFirst(ClaimTypes.Gender)?.Value;
        var avatarUrl = "";

        var platformType = PlatformTypeEnum.微信公众号;
        if (provider == "Gitee")
        {
            platformType = PlatformTypeEnum.Gitee;
            avatarUrl = authenticateResult.Principal.FindFirst(OAuthClaim.GiteeAvatarUrl)?.Value;
        }

        // 若账号不存在则新建
        var wechatUser = await _sysOAuthUserRep.AsQueryable().Includes(u => u.SysUser).ClearFilter().FirstAsync(u => u.OpenId == openIdClaim.Value);
        if (wechatUser == null)
        {
            var userId = await App.GetRequiredService<SysUserService>().AddUser(new AddUserInput()
            {
                Account = name,
                RealName = name,
                NickName = name,
                Email = email,
                Avatar = avatarUrl,
                Phone = mobilePhone,
                OrgId = 1300000000101, // 根组织架构
                RoleIdList = new List<long> { 1300000000104 } // 仅本人数据角色
            });

            await _sysOAuthUserRep.InsertAsync(new SysOAuthUser()
            {
                UserId = userId,
                OpenId = openIdClaim.Value,
                Avatar = avatarUrl,
                NickName = name,
                PlatformType = platformType
            });

            wechatUser = await _sysOAuthUserRep.AsQueryable().Includes(u => u.SysUser).ClearFilter().FirstAsync(u => u.OpenId == openIdClaim.Value);
        }

        // 构建Token令牌
        var token = await App.GetRequiredService<SysAuthService>().CreateToken(wechatUser.SysUser);

        return new RedirectResult($"{redirectUrl}/#/login?token={token.AccessToken}");
    }
}
```