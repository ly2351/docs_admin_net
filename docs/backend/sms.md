---
sidebar_position: 15
---

# 15. 短信发送

:::tip 提示

Admin.NET 采用 AlibabaCloud.SDK.Dysmsapi20170525 阿里云短信的SDK组件。访问密钥（AccessKey）及访问 Id 的获取请参考阿里云短信jicheng
:::

短信配置文件 Configuration/SMS.json，集成教程 阿里云集成SDK

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "SMS": {
    "Aliyun": {
      "AccessKeyId": "",
      "AccessKeySecret": "",
      "SignName": "AdminNET平台", // 短信签名
      "TemplateCode": "" // 短信模板
    }
  }
}
```

系统继承实现的具体类文件 Admin.NET.Core/Service/Message/SysSmsService.cs

```csharp
using AlibabaCloud.SDK.Dysmsapi20170525.Models;

namespace Admin.NET.Core.Service;

/// <summary>
/// 系统短信服务 🧩
/// </summary>
[AllowAnonymous]
[ApiDescriptionSettings(Order = 150)]
public class SysSmsService : IDynamicApiController, ITransient
{
    private readonly SMSOptions _smsOptions;
    private readonly SysCacheService _sysCacheService;

    public SysSmsService(IOptions<SMSOptions> smsOptions,
        SysCacheService sysCacheService)
    {
        _smsOptions = smsOptions.Value;
        _sysCacheService = sysCacheService;
    }

    /// <summary>
    /// 发送短信 📨
    /// </summary>
    /// <param name="phoneNumber"></param>
    /// <returns></returns>
    [AllowAnonymous]
    [DisplayName("发送短信")]
    public async Task SendSms([Required] string phoneNumber)
    {
        if (!phoneNumber.TryValidate(ValidationTypes.PhoneNumber).IsValid)
            throw Oops.Oh("请正确填写手机号码");

        // 生成随机验证码
        var random = new Random();
        var verifyCode = random.Next(100000, 999999);

        var templateParam = Clay.Object(new
        {
            code = verifyCode
        });

        var client = CreateClient();
        var sendSmsRequest = new SendSmsRequest
        {
            PhoneNumbers = phoneNumber, // 待发送手机号, 多个以逗号分隔
            SignName = _smsOptions.Aliyun.SignName, // 短信签名
            TemplateCode = _smsOptions.Aliyun.TemplateCode, // 短信模板
            TemplateParam = templateParam.ToString(), // 模板中的变量替换JSON串
            OutId = YitIdHelper.NextId().ToString()
        };
        var sendSmsResponse = client.SendSms(sendSmsRequest);
        if (sendSmsResponse.Body.Code == "OK" && sendSmsResponse.Body.Message == "OK")
        {
            // var bizId = sendSmsResponse.Body.BizId;
            _sysCacheService.Set($"{CacheConst.KeyPhoneVerCode}{phoneNumber}", verifyCode, TimeSpan.FromSeconds(60));
        }
        else
        {
            throw Oops.Oh($"短信发送失败：{sendSmsResponse.Body.Code}-{sendSmsResponse.Body.Message}");
        }

        await Task.CompletedTask;
    }

    /// <summary>
    /// 阿里云短信配置
    /// </summary>
    /// <returns></returns>
    private AlibabaCloud.SDK.Dysmsapi20170525.Client CreateClient()
    {
        var config = new AlibabaCloud.OpenApiClient.Models.Config
        {
            AccessKeyId = _smsOptions.Aliyun.AccessKeyId,
            AccessKeySecret = _smsOptions.Aliyun.AccessKeySecret,
            Endpoint = "dysmsapi.aliyuncs.com"
        };
        return new AlibabaCloud.SDK.Dysmsapi20170525.Client(config);
    }
}
```

:::tip 提示

每次发送短信时系统会自动缓存手机验证码，默认 60 秒有效期，供手机号登录验证等场景模式。直接使用方法 SendSms 指定手机号即可发送短信。
:::

下面是系统自带的手机号登录场景模式，实现类为 Admin.NET.Core/Service/Auth/SysAuthService.cs。

```csharp
    /// <summary>
    /// 手机号登录 🔖
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    [AllowAnonymous]
    [DisplayName("手机号登录")]
    public virtual async Task<LoginOutput> LoginPhone([Required] LoginPhoneInput input)
    {
        var verifyCode = _sysCacheService.Get<string>($"{CacheConst.KeyPhoneVerCode}{input.Phone}");
        if (string.IsNullOrWhiteSpace(verifyCode))
            throw Oops.Oh("验证码不存在或已失效，请重新获取！");
        if (verifyCode != input.Code)
            throw Oops.Oh("验证码错误！");

        // 账号是否存在
        var user = await _sysUserRep.AsQueryable().Includes(t => t.SysOrg).ClearFilter().FirstAsync(u => u.Phone.Equals(input.Phone));
        _ = user ?? throw Oops.Oh(ErrorCodeEnum.D0009);

        return await CreateToken(user);
    }
```    