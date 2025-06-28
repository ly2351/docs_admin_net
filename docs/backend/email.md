---
sidebar_position: 18
---

# 18. 邮件发送

:::tip 提示

邮件处理采用 MailKit 组件框架。MailKit 是一个开源的 C# 邮件处理库，用于在应用程序中发送和接收电子邮件。它提供了一个强大且易于使用的 API，支持多种邮件协议，包括 SMTP、POP3、和 IMAP。
:::

## SMTP协议
SMTP是一种提供可靠且有效的电子邮件传输的协议。SMTP是建立在FTP文件传输服务上的一种邮件服务，主要用于系统之间的邮件信息传递，并提供有关来信的通知。SMTP独立于特定的传输子系统，且只需要可靠有序的数据流信道支持，SMTP的重要特性之一是其能跨越网络传输邮件，即“SMTP邮件中继”。使用SMTP，可实现相同网络处理进程之间的邮件传输，也可通过中继器或网关实现某处理进程与其他网络之间的邮件传输。

特性与优势 描述 多协议支持 支持 SMTP、POP3、IMAP 等多种邮件协议。 异步操作 使用异步编程模型，提高性能和响应性。 附件处理 提供灵活的附件处理功能，支持添加、读取和保存邮件附件。 SSL/TLS 支持 支持安全套接字层（SSL）和传输层安全性（TLS），确保邮件的安全传输。 容错处理 提供容错处理机制，处理网络或协议错误，确保稳定的邮件通信。 丰富的 API 提供丰富的 API，方便开发人员访问邮件的各个方面，包括主题、发件人、收件人等。 跨平台 MailKit 是一个跨平台的邮件处理库，可在多个操作系统上运行，包括 Windows、Linux 和 macOS。

| 邮箱类型       | POP3服务器                          | SMTP服务器                             |
|----------------|-------------------------------------|-----------------------------------------|
| 【QQ邮箱】     | pop.qq.com（端口：110）            | smtp.qq.com（端口：25）                |
| 【sina.com】   | pop3.sina.com.cn（端口：110）      | smtp.sina.com.cn（端口：25）          |
| 【sinaVIP】    | pop3.vip.sina.com（端口：110）     | smtp.vip.sina.com（端口：25）         |
| 【sohu.com】   | pop3.sohu.com（端口：110）         | smtp.sohu.com（端口：25）             |
| 【126邮箱】    | pop.126.com（端口：110）           | smtp.126.com（端口：25）              |
| 【139邮箱】    | POP.139.com（端口：110）           | SMTP.139.com（端口：25）              |
| 【163.com】    | pop.163.com（端口：110）           | smtp.163.com（端口：25）              |
| 【QQ企业邮箱】 | pop.exmail.qq.com（SSL启用 端口：995） | smtp.exmail.qq.com（SSL启用 端口：587/465） |
| 【yahoo.com】  | pop.mail.yahoo.com                 | smtp.mail.yahoo.com                    |
| 【yahoo.com.cn】| pop.mail.yahoo.com.cn（端口：995）| smtp.mail.yahoo.com.cn（端口：587）   |
| 【HotMail】    | pop3.live.com（端口：995）         | smtp.live.com（端口：587）            |
| 【Gmail】      | pop.gmail.com（SSL启用端口：995）  | smtp.gmail.com（SSL启用 端口：587）   |
| 【263.net】    | pop3.263.net（端口：110）          | smtp.263.net（端口：25）              |
| 【263.net.cn】 | pop.263.net.cn（端口：110）        | smtp.263.net.cn（端口：25）           |
| 【21cn.com】   | pop.21cn.com（端口：110）          | smtp.21cn.com（端口：25）             |
| 【Foxmail】    | POP.foxmail.com（端口：110）       | SMTP.foxmail.com（端口：25）          |
| 【china.com】  | pop.china.com（端口：110）         | smtp.china.com（端口：25）            |
| 【tom.com】    | pop.tom.com（端口：110）           | smtp.tom.com（端口：25）              |


:::tip 提示

Admin.NET 全局异常日志默认启动发送到指定邮箱。开关由系统参数配置关键字 CommonConst.SysErrorMail 来控制。
:::

## 配置文件

邮箱配置文件 Configuration/Email.json

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "Email": {
    "Host": "smtp.163.com", // 主机
    "Port": 465, // 端口 465、994、25
    "EnableSsl": true, // 启用SSL
    "DefaultFromEmail": "xxx@163.com", // 默认发件者邮箱
    "DefaultToEmail": "xxx@qq.com", // 默认接收人邮箱
    "UserName": "xxx@163.com", // 邮箱账号
    "Password": "", // 邮箱授权码
    "DefaultFromName": "Admin.NET 通用权限开发平台" // 默认邮件标题
  }
}
```

## 系统实现

```csharp
public class SysEmailService : IDynamicApiController, ITransient
{
    private readonly EmailOptions _emailOptions;

    public SysEmailService(IOptions<EmailOptions> emailOptions)
    {
        _emailOptions = emailOptions.Value;
    }

    /// <summary>
    /// 发送邮件 📧
    /// </summary>
    /// <param name="content"></param>
    /// <param name="title"></param>
    /// <returns></returns>
    [DisplayName("发送邮件")]
    public async Task SendEmail([Required] string content, string title = "Admin.NET 系统邮件")
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress(_emailOptions.DefaultFromEmail, _emailOptions.DefaultFromEmail));
        message.To.Add(new MailboxAddress(_emailOptions.DefaultToEmail, _emailOptions.DefaultToEmail));
        message.Subject = title;
        message.Body = new TextPart("html")
        {
            Text = content
        };

        using var client = new SmtpClient();
        client.Connect(_emailOptions.Host, _emailOptions.Port, _emailOptions.EnableSsl);
        client.Authenticate(_emailOptions.UserName, _emailOptions.Password);
        client.Send(message);
        client.Disconnect(true);

        await Task.CompletedTask;
    }
}
```

## 使用方式

首先注入邮件服务 `SysEmailService`，然后调用方法 `SendEmail` 即可，邮件内容默认 HTML 格式。建议发送邮件、短信等消息提醒时采用事件总线模式，不阻塞主线程运行。

## 邮件模板

```csharp
[EventSubscribe("Mail:SendOrderMail")]
public async Task SendOrderMail(EventHandlerExecutingContext context)
{
    // var orderId = (long)context.Source.Payload; // 从事件总线传过来的值

    var mailTempPath = Path.Combine(App.WebHostEnvironment.WebRootPath, "Temp\\ErrorMail.tp");
    var mailTemp = File.ReadAllText(mailTempPath);
    var mail = await _serviceScope.ServiceProvider.GetRequiredService<IViewEngine>().RunCompileFromCachedAsync(mailTemp, 数据内容);

    var title = "Admin.NET 系统异常";
    await _serviceScope.ServiceProvider.GetRequiredService<SysEmailService>().SendEmail(mail, title);
}
```

下面是一个 HTML 邮件模板，直接采用视图模板引擎 IViewEngine 自动替换模板数据。

```html
<!DOCTYPE html>

<html lang="en" xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="utf-8" />
    <title>订单邮件模板</title>
    <style>
        .byTable {
            width: 100%;
            border-collapse: collapse;
            padding: 2px;
        }

        .byTable, .byTable tr th, .byTable tr td {
            border: 1px solid black;
            padding: 5px 15px;
            font-size: 16px;
            }

        .tdbg1 {
            background-color: #EEEEEE;
            font-weight: 700;
        }

        .tdbg2 {
            background-color: #EEEEEE;
        }

        .tdbg3 {
            font-weight: 700;
        }
    </style>
</head>
<body>
    <h4>订单详情</h4>
    <table class="byTable">
        <tr>
            <td class="tdbg1">订单号</td>
            <td class="tdbg2">@Model.OrderId</td>
            <td class="tdbg1">销售商单号</td>
            <td class="tdbg2">@Model.SellerOrderId</td>
            <td class="tdbg1">供应商单号</td>
            <td class="tdbg2">@Model.SupplierOrderId</td>
        </tr>
        <tr>
            <td class="tdbg1">订单时间</td>
            <td class="tdbg2">@Model.OrderTime</td>
            <td class="tdbg1">入住日期</td>
            <td class="tdbg2">@Model.CheckInTime</td>
            <td class="tdbg1">离店日期</td>
            <td class="tdbg2">@Model.CheckOutTime</td>
        </tr>
        <tr>
            <td class="tdbg1">预定人员</td>
            <td class="tdbg2">@Model.Booker</td>
            <td class="tdbg1">手机号码</td>
            <td class="tdbg2">@Model.Phone</td>
            <td class="tdbg1">房间数量</td>
            <td class="tdbg2">@Model.Rooms</td>
        </tr>
        <tr></tr>
        <tr>
            <td class="tdbg3">销售商名称</td>
            <td>@Model.SellerName</td>
            <td class="tdbg3">基础酒店名称</td>
            <td>@Model.BaseHotelName</td>
            <td class="tdbg3">基础房型编码</td>
            <td>@Model.BaseRoomCode</td>
        </tr>
        <tr>
            <td class="tdbg3">销售商酒店地址</td>
            <td colspan="5">@Model.SellerHotelAddress</td>
        </tr>
        <tr>
            <td class="tdbg3">供应商名称</td>
            <td>@Model.SupplierName</td>
            <td class="tdbg3">供应商酒店名称</td>
            <td>@Model.SupplierHotelName</td>
            <td class="tdbg3">供应商房型编码</td>
            <td>@Model.SupplierRoomCode</td>
        </tr>
        <tr>
            <td class="tdbg3">供应商酒店地址</td>
            <td colspan="5">@Model.SupplierHotelAddress</td>
        </tr>
        <tr></tr>
        <tr>
            <td class="tdbg3">供应商餐饮类型</td>
            <td>@Model.SupplierBreakfastBoardCode</td>
            <td class="tdbg3">供应商餐饮数量</td>
            <td>@Model.SupplierBreakfastCount</td>
            <td></td>
            <td></td>
        </tr>
    </table>
</body>
</html>
```