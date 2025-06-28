---
sidebar_position: 16
---

# 16. 微信对接

:::tip 提示

基于 Flurl.Http SKIT.FlurlHttpClient.Wechat 的微信 HTTP API SDK，目前已包含包含微信公众平台（订阅号+服务号+小程序+小游戏+小商店+视频号）、微信开放平台、微信商户平台（微信支付+微企付）、企业微信、微信广告平台、微信智能对话开放平台等模块。默认是 V3 版本。
:::

:::info 相关信息

- 本库专注于 API 本身的封装，捎带提供了一些用于加解密、序列化的工具类，使用起来更加灵活，不限任何框架或项目类型；盛派微信 SDK 提供了大而全的功能，与 MVC / WebAPI 深度集成。

- 本库的接口模型遵循的是微软官方推荐的 C# 属性命名方式（帕斯卡命名法）；盛派微信 SDK 提供的是微信接口本身的命名方式（蛇形命名法和驼峰命名法混杂）。

- 本库封装了目前微信官方提供的几乎所有 API（极个别不支持的已在各模块文档中列出具体原因）；盛派微信 SDK 只提供了常用的 API。
:::

本库提供的请求模型、响应模型和接口方法，三者均保持同名。例如，发送模板消息的请求是 CgibinMessageTemplateSendRequest，响应是 CgibinMessageTemplateSendResponse，接口是 ExecuteCgibinMessageTemplateSendAsync()。知道其中一个，其余两个就可以快速地推断出了。再有，每个对象的命名与官方文档的接口地址大体保持一致。例如刚刚提到的发送模板消息，它的接口地址是 [POST] /cgi-bin/message/template/send，将其中的反斜杠去掉、并以大驼峰命名法的方式调整它，就可以得到前文提到的几个对象了。完整的模型定义可以参考项目目录下的 src/SKIT.FlurlHttpClient.Wechat.Api/Models 目录。

微信相关配置文件 Configuration/Wechat.json，具体实现类 Admin.NET.Core/Service/Wechat

```json
{
  "$schema": "https://gitee.com/dotnetchina/Furion/raw/v4/schemas/v4/furion-schema.json",

  "Wechat": {
    // 公众号
    "WechatAppId": "",
    "WechatAppSecret": "",
    "WechatToken": "", // 微信公众号服务器配置中的令牌(Token)
    "WechatEncodingAESKey": "", // 微信公众号服务器配置中的消息加解密密钥(EncodingAESKey)
    // 小程序
    "WxOpenAppId": "",
    "WxOpenAppSecret": "",
    "WxToken": "", // 小程序消息推送中的令牌(Token)
    "WxEncodingAESKey": "" // 小程序消息推送中的消息加解密密钥(EncodingAESKey)
  },
  // 微信支付
  "WechatPay": {
    "AppId": "", // 微信公众平台AppId、开放平台AppId、小程序AppId、企业微信CorpId
    "MerchantId": "", // 商户平台的商户号
    "MerchantV3Secret": "", // 商户平台的APIv3密钥
    "MerchantCertificateSerialNumber": "", // 商户平台的证书序列号
    "MerchantCertificatePrivateKey": "\\WxPayCert\\apiclient_key.pem" // 商户平台的API证书私钥(apiclient_key.pem文件内容)
  },
  // 支付回调
  "PayCallBack": {
    "WechatPayUrl": "https://xxx/api/sysWechatPay/payCallBack", // 微信支付回调
    "WechatRefundUrl": "", // 微信退款回调
    "AlipayUrl": "", // 支付宝支付回调
    "AlipayRefundUrl": "" // 支付宝退款回调
  }
}
```

:::tip提示

Admin.NET 已经继承微信公众号登录、获取用户信息、支付等逻辑，微信小程序登录、获取用户信息、消息模板及发送消息等功能。已经和第三方登录授权逻辑及三方账号表关联，可参考 /后端手册/8统一认证。
:::

使用过程中，其实拿到微信对象 WechatApiClient ，理论上可以操作所有所有的微信接口，业务应用层所需微信接口自行扩展即可。具体使用教程 [使用文档](https://gitee.com/fudiwei/DotNetCore.SKIT.FlurlHttpClient.Wechat/tree/main/docs)

