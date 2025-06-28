---
sidebar_position: 2
---

# 2. 开发流程

## 如何开发
:::info 相关信息

1、首选 fork 代码到自己的仓库（也可以下载最新的代码版本到自己本地，后续进行对比同步升级）

2、新增一个分支开发应用业务 比如：xxx。主分支一般是master用来拉取同步最新代码(会被强制覆盖)，不要开发自己的业务

3、在此代码解决方案里面，新建一个业务应用工程，比如：xxx.Application

4、去掉 Admin.NET.Web.Core 工程默认引用的 Admin.NET.Application 应用，更换成自己的业务应用层工程 xxx.Application

5、将默认应用工程 Admin.NET.Application 里面的配置文件夹 Configuration 复制一份到自己的业务应用层工程 xxx.Application

6、参考 Admin.NET.Application 工程文件设置修改新建的业务应用层工程 xxx.Application 工程文件

7、若业务应用比较独立，可以直接创建插件工程，然后将插件工程引用到自己的应用工程里面来

8、尽量保持底层框架代码不变，方便同步升级，减少更新冲突

9、之后通过同步拉取到的最新代码合并到自己的 xxx 分支上
:::

:::tip 提示

推荐新建一个业务应用工程，比如：xxx.Application，去掉 Admin.NET.Web.Core 工程默认引用的 Admin.NET.Application 应用，更换成自己的业务应用层工程 xxx.Application。
:::

## 配置文件

将应用工程 Admin.NET.Application 里面的配置文件夹 Configuration 复制一份到自己的业务应用层工程 xxx.Application 里面。`可以在此文件夹里面任意添加业务应用相关的配置选项。`

## 工程文件
双击新建的应用工程，弹出工程文件进入编辑页面，添加 Configuration 配置文件夹自动复制，免去每新增一个配置选项文件就得右键设置更新复制，又简化了工程文件内容。添加如下即可：
```csharp
  <ItemGroup>
    <Content Include="Configuration\**\*">
      <ExcludeFromSingleFile>true</ExcludeFromSingleFile>
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
    </Content>
    <Content Include="wwwroot\**\*">
      <ExcludeFromSingleFile>true</ExcludeFromSingleFile>
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
    </Content>
  </ItemGroup>
```
:::tip 提示

配置 wwwroot 复制，可以放些与业务应用相关的静态文件，发布的时候文件夹自动合并，比如报告模板、导入导出模板等。
:::

若希望当前工程支持多平台模式，比如同时支持 .NET6 和 .NET8 , 则双击工程文件进入编辑页面，在各自平台下引用不同版本的nuget包，添加如下：
```csharp
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net6.0;net8.0</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup>

  </ItemGroup>
 
  <ItemGroup Condition=" '$(TargetFramework)' == 'net6.0' ">
  
  </ItemGroup>

  <ItemGroup Condition=" '$(TargetFramework)' == 'net8.0' ">

  </ItemGroup>

</Project>
```

## 静态变量
推荐新建一个 Const 文件夹，里面放业务应用相关的全局静态变量，比如接口分组名称等，这样定义接口的时候就可以直接用此变量定义接口分组了。

```csharp
/// <summary>
/// 业务应用相关常量
/// </summary>
public class ApplicationConst
{
    /// <summary>
    /// API分组名称
    /// </summary>
    public const string GroupName = "xxx业务应用";
}
```

## 实体定义
:::tip 提示

实体定义完成，启动服务并且数据库配置文件里面Database.json已启用表初始化开关 "EnableInitTable": true, // 启用表初始化，则自动会创建数据表。每次修改实体代码，也会自动同步数据库表结构。

实体基类请参考说明 实体基类
:::

推荐新建一个 Entity 文件夹，里面放业务应用相关的实体类（数据表）。也可以按具体业务应用分多级目录放置实体类。`若业务实体很多时，也可以放置在各个服务文件夹里面，此时服务文件夹包括接口实现和实体定义。` 下面是实体定义的示例，供参考：
```csharp
/// <summary>
/// xxx业务表
/// </summary>
[SugarTable(null, "xxx业务表")]
public partial class xxx : EntityTenant
{
    /// <summary>
    /// 名称
    /// </summary>
    [SugarColumn(ColumnDescription = "名称", Length = 64)]
    [Required, MaxLength(64)]
    public virtual string Name { get; set; }

    /// <summary>
    /// 编码
    /// </summary>
    [SugarColumn(ColumnDescription = "编码", Length = 64)]
    [MaxLength(64)]
    public string? Code { get; set; }

    /// <summary>
    /// 排序
    /// </summary>
    [SugarColumn(ColumnDescription = "排序")]
    public int OrderNo { get; set; } = 100;

    /// <summary>
    /// 备注
    /// </summary>
    [SugarColumn(ColumnDescription = "备注", Length = 128)]
    [MaxLength(128)]
    public string? Remark { get; set; }

    /// <summary>
    /// 状态
    /// </summary>
    [SugarColumn(ColumnDescription = "状态")]
    public StatusEnum Status { get; set; } = StatusEnum.Enable;
}
```
:::tip 提示

1、实体定义继承的基类推荐框架自带的。（框架会自动处理基类实体的字段赋值操作）

2、字符串类型字符，最好定义长度，推荐长度为 2 的指数。若此字段为空，只需在 string 后面添加问号 ? 即可。

3、定义实体表特性 [SugarTable(null, "xxx业务表")] 时， 推荐第一个参数为 null，可以方便适配各种数据库表名命名规范。
:::

## 接口服务（WebAPI）
:::tip 提示

Admin.NET 接口服务实现集中一个文件实现，即接口定义、控制器、接口实现都在一个类里面。也可以把控制器和接口实现分开。所有的接口服务、实体类、定时任务、事件订阅等等实现，都不需要手动注入，框架都是自动遍历全局进行自动注入。

1、接口服务名称默认去除以 AppServices，AppService，ApiController，Controller，Services，Service 作为前后缀的字符串。比如 AdminService -> Admin 。

2、接口服务名称带 V[0-9_] 结尾的，会自动生成控制器版本号，如 AdminServiceV2 -> Admin@2，AdminServiceV1_1_0 -> Admin@1.1.0。

3、方法名称默认去除以 Post/Add/Create/Insert/Submit，GetAll/GetList/Get/Find/Fetch/Query/Search，Put/Update，Delete/Remove/Clear，Patch 开头的字符串；默认去除以 Async 作为前后缀的字符串。

4、方法名称带 V[0-9_] 结尾的，会自动生成动作方法版本号，如 AdminV2 -> Admin@2，AdminV1_1_0 -> Admin@1.1.0。

5、请求默认添加 [HttpPost] 特性。请求谓词约定1、以 Post/Add/Create/Insert/Submit/Change 开头，则添加 [HttpPost] 特性 2、以 GetAll/GetList/Get/Find/Fetch/Query 开头，则添加 [HttpGet] 特性 3、以 Put/Update 开头，则添加 [HttpPut] 特性 4、以 Delete/Remove/Clear 开头，则添加 [HttpDelete] 特性 5、以 Patch 开头，则添加 [HttpPatch] 特性。 推荐只用 HttpGet 和 HttpPost 两种。

6、路由地址前缀默认以 api 打头。

7、建议每个公开方法/WebAPI定义上面特特性 [DisplayName("xxx接口名字")]，这样在操作日志里面可以查询统计每个接口名称。
:::

推荐新建一个 Service 文件夹，里面放业务应用相关的接口服务。也可以按具体业务应用分多级目录放置。下面是接口服务定义的示例，供参考：

```csharp
/// <summary>
/// xxx 服务
/// </summary>
[ApiDescriptionSettings(ApplicationConst.GroupName, Order = 100)]
public class xxxService : IDynamicApiController, ITransient
{
    private readonly SqlSugarRepository<xxx> _xxxRep;

    public xxxService(SqlSugarRepository<xxx> xxxRep)
    {
        _xxxRep = xxxRep;
    }
}
```

1、接口服务类直接继承动态 WebAPI 控制器接口 IDynamicApiController 即可，这样类里面的 public 方法就是一个 API 接口，简单吧 😎。若无需显示某个公开方法或控制器，只需要添加 [ApiDescriptionSettings(false)] 或 [ApiDescriptionSettings(IgnoreApi = true)] 即可。公开方法还支持特性 [NonAction]，即可将一个公开方法不生成 API 接口。

2、接口服务 3 个不同的依赖接口：1、ITransient：对应暂时/瞬时作用域服务生存期 2、IScoped：对应请求作用域服务生存期 3、ISingleton：对应单例作用域服务生存期，只能实例类实现，其他静态类、抽象类、及接口不能实现，请根据业务需要自行选择继承。只要任意继承某个服务生存期，该接口类自动注入服务，无需再手动添加注入。

3、接口服务分组、排序等定义 [ApiDescriptionSettings(ApplicationConst.GroupName, Order = 100)]，其中 ApplicationConst.GroupName 定义接口分组名称，Order 为接口分组排序。

4、实体/表操作只需要注入该实体仓储即可 如 `private readonly SqlSugarRepository<xxx> _xxxRep`, 以 _xxxRep 就可以对表进行增删改查了。具体教程可以参考 https://www.donet5.com/Home/Doc?typeId=1228。

5、其他功能类服务相同，想调用哪个服务，必须先注入再使用。或者使用 `App.GetRequiredService<xxxService>();` 模式直接获取服务进行操作。

:::tip 提示

推荐在每个接口服务文件夹下面创建一个 DTO 文件夹，里面放置与前端交互的入参与返回结果的数据结构文件。入参推荐以 Input 结尾，返回结果以 Output 结尾。

下面是一个完整接口服务类定义（通知公告接口服务类），包括分页查询、增、删、改、查具体操作：
:::
```csharp
/// <summary>
/// 系统通知公告服务 🧩
/// </summary>
[ApiDescriptionSettings(Order = 380)]
public class SysNoticeService : IDynamicApiController, ITransient
{
    private readonly UserManager _userManager;
    private readonly SqlSugarRepository<SysUser> _sysUserRep;
    private readonly SqlSugarRepository<SysNotice> _sysNoticeRep;
    private readonly SqlSugarRepository<SysNoticeUser> _sysNoticeUserRep;
    private readonly SysOnlineUserService _sysOnlineUserService;

    public SysNoticeService(
        UserManager userManager,
        SqlSugarRepository<SysUser> sysUserRep,
        SqlSugarRepository<SysNotice> sysNoticeRep,
        SqlSugarRepository<SysNoticeUser> sysNoticeUserRep,
        SysOnlineUserService sysOnlineUserService)
    {
        _userManager = userManager;
        _sysUserRep = sysUserRep;
        _sysNoticeRep = sysNoticeRep;
        _sysNoticeUserRep = sysNoticeUserRep;
        _sysOnlineUserService = sysOnlineUserService;
    }

    /// <summary>
    /// 获取通知公告分页列表 📢
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    [DisplayName("获取通知公告分页列表")]
    public async Task<SqlSugarPagedList<SysNotice>> Page(PageNoticeInput input)
    {
        return await _sysNoticeRep.AsQueryable()
            .WhereIF(!string.IsNullOrWhiteSpace(input.Title), u => u.Title.Contains(input.Title.Trim()))
            .WhereIF(input.Type > 0, u => u.Type == input.Type)
            .WhereIF(!_userManager.SuperAdmin, u => u.CreateUserId == _userManager.UserId)
            .OrderBy(u => u.CreateTime, OrderByType.Desc)
            .ToPagedListAsync(input.Page, input.PageSize);
    }

    /// <summary>
    /// 增加通知公告 📢
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    [ApiDescriptionSettings(Name = "Add"), HttpPost]
    [DisplayName("增加通知公告")]
    public async Task AddNotice(AddNoticeInput input)
    {
        var notice = input.Adapt<SysNotice>();
        InitNoticeInfo(notice);
        await _sysNoticeRep.InsertAsync(notice);
    }

    /// <summary>
    /// 更新通知公告 📢
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    [UnitOfWork]
    [ApiDescriptionSettings(Name = "Update"), HttpPost]
    [DisplayName("更新通知公告")]
    public async Task UpdateNotice(UpdateNoticeInput input)
    {
        var notice = input.Adapt<SysNotice>();
        InitNoticeInfo(notice);
        await _sysNoticeRep.UpdateAsync(notice);
    }

    /// <summary>
    /// 删除通知公告 📢
    /// </summary>
    /// <param name="input"></param>
    /// <returns></returns>
    [UnitOfWork]
    [ApiDescriptionSettings(Name = "Delete"), HttpPost]
    [DisplayName("删除通知公告")]
    public async Task DeleteNotice(DeleteNoticeInput input)
    {
        await _sysNoticeRep.DeleteAsync(u => u.Id == input.Id);

        await _sysNoticeUserRep.DeleteAsync(u => u.NoticeId == input.Id);
    }
}
```
:::tip 提示

其中特性 [UnitOfWork] 代表数据库单元事务，处理一个 webAPI 方法内同时操作多张表时，保持数据一致性。注意此单元事务只在 WebAPI 模式下生效，普通方法无效。 其中 [ApiDescriptionSettings(Name = "Delete"), HttpPost] 是强制指定接口名称的，保持 Delete 字符串存在接口路由中，并指定 HttpPost 请求模式。其他类同。
:::
