---
sidebar_position: 1
---

# 1. 框架简介

:::tip 项目整体目录结构图介绍
Admin.NET 项目主要包括 4 个工程，应用层 `Admin.NET.Application`、核心层 `Admin.NET.Core`、Web 核心层 `Admin.NET.Web.Core`、启动层 `Admin.NET.Web.Entry`。
:::

```xml

├── Admin.NET.Application(业务应用层，放置具体业务实体及接口实现)【引用 `Admin.NET.Core` 工程】
    ├── Configuration (业务应用各种配置文件)
        │── APIJSON.json (APIJSON配置)
        │── App.json (框架配置)
        │── Cache.json (缓存配置)
        │── Captcha.json (验证码配置)
        │── CodeGen.json (代码生成配置)
        │── Database.json (数据库配置)
        │── Email.json (邮件配置)
        │── Enum.json (枚举配置)
        │── EventBus.json (事件配置)
        │── JWT.json (Token配置)
        │── Limit.json (接口限流配置)
        │── Logging.json (日志配置)
        │── OAuth.json (统一授权配置)
        │── SMS.json (短信配置)
        │── Swagger.json (Swagger配置)
        │── Upload.json (文件上传配置)
        │── Wechat.json (微信相关配置)
    │── Const (业务应用常量)
    ├── Service (业务应用接口服务类)
    │── Startup.cs (业务应用启动类)

├── Admin.NET.Core(框架核心层) 【只需要让应用层引用】
    ├── Attribute (特性定义)
    ├── Cache (缓存管理)
    ├── Const (框架常量)
    ├── Entity (框架实体)
    ├── Enum (框架枚举)
    ├── EventBus (事件总线实现)
    ├── Extension (常用扩展)
    ├── Hub (即时通讯/集线器)
    ├── Job (定时任务)
    ├── Logging (日志)
    ├── Option (配置选项)
    ├── SeedData (种子数据)
    ├── Service (所有接口服务)
        │── APIJSON (APIJSON)
        │── Auth (登录授权)
        │── Cache (缓存管理)
        │── CodeGen (代码生成)
        │── Common (常用接口)
        │── Config (系统参数配置)
        │── Const (系统常量)
        │── DataBase (数据库管理)
        │── Dict (字典管理)
        │── Enum (枚举管理)
        │── File (文件上传下载)
        │── Job (定时任务)
        │── Logging (系统日志)
        │── Menu (菜单管理)
        │── Message (消息管理)
        │── Notice (公告通知)
        │── OAuth (统一登录授权)
        │── OnlineUser (在线用户)
        │── OpenAccess (开放接口)
        │── Org (组织架构)
        │── Plugin (插件管理)        
        │── Pos (职位管理)
        │── Print (模板打印)
        │── Region (行政区域)
        │── Role (系统角色)
        │── Server (服务器相关)
        │── Tenant (租户管理)
        │── User (账号管理)
        │── Wechat (微信相关)
        │── BaseService.cs (实体操作基服务)
    ├── SignalR (SignalR 初始化)
    ├── SignatureAuth (数字签名)
    ├── SqlSugar (ORM 封装)
    ├── Util (常用工具类)

├── Admin.NET.Web.Core(Web 核心层) 【引用 `Admin.NET.Application` 工程】
    ├── Handlers (鉴权授权)
        │── JwtHandler.cs (鉴权授权实现)
    ├── ProjectOptions.cs (注册项目配置选项)
    ├── Startup.cs (项目启动类，所有组件在此注册引入)

├── Admin.NET.Web.Entry(项目启动层) 【引用 `Admin.NET.Web.Core` 工程】
    ├── sensitive-words.txt (脱敏词汇)
    ├── SingleFilePublish.cs (单文件发布配置)
    ├── ip2region.db (IP 地址库)
    ├── GeoLite2-City.mmdb (IP 地址库)
    ├── ......
```