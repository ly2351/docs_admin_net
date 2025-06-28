---
sidebar_position: 3
---

# 3. 实体基类

:::tip 提示

框架自带了 6 种实体基类，自己定义业务实体时可以自行选择继承。框架默认所有表 Id 为雪花Id，长整型。雪花算法中非常好用的数字ID生成器 https://github.com/yitter/idgenerator。
:::

雪花Id配置见 /Configuration/App.json
```csharp
  "SnowId": {
    "WorkerId": 1, // 机器码 全局唯一
    "WorkerIdBitLength": 6, // 机器码位长 默认值6，取值范围 [1, 19]
    "SeqBitLength": 6, // 序列数位长 默认值6，取值范围 [3, 21]（建议不小于4，值越大性能越高、Id位数也更长）
    "WorkerPrefix": "adminnet_" // 缓存前缀
  },
```
## 1、只包括雪花 Id
:::tip 提示

新增时，若Id无值，系统自动赋值雪花Id；若有值，则不处理。

:::
```csharp
/// <summary>
/// 框架实体基类Id
/// </summary>
public abstract class EntityBaseId
{
    /// <summary>
    /// 雪花Id
    /// </summary>
    [SugarColumn(ColumnName = "Id", ColumnDescription = "主键Id", IsPrimaryKey = true, IsIdentity = false)]
    public virtual long Id { get; set; }
}
```

## 2、包括创建时间、创建者、更新时间、更新者、软删除
:::tip 提示

新增时，若创建时间无值，系统自动赋值当前时间；若有值，则不处理，新增时不处理更新时间、更新者字段（除非已主动赋值）。更新时，若更新时间无值，系统自动赋值当前时间；若有值，则不处理。若是 web 线程时，系统自动处理创建者和更新者字段值。应用层无需手动处理。
:::
```csharp
/// <summary>
/// 框架实体基类
/// </summary>
[SugarIndex("index_{table}_CT", nameof(CreateTime), OrderByType.Asc)]
public abstract class EntityBase : EntityBaseId, IDeletedFilter
{
    /// <summary>
    /// 创建时间
    /// </summary>
    [SugarColumn(ColumnDescription = "创建时间", IsOnlyIgnoreUpdate = true, InsertServerTime = true)]
    public virtual DateTime CreateTime { get; set; }

    /// <summary>
    /// 更新时间
    /// </summary>
    [SugarColumn(ColumnDescription = "更新时间", IsOnlyIgnoreInsert = true, UpdateServerTime = true)]
    public virtual DateTime? UpdateTime { get; set; }

    /// <summary>
    /// 创建者Id
    /// </summary>
    [SugarColumn(ColumnDescription = "创建者Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? CreateUserId { get; set; }

    ///// <summary>
    ///// 创建者
    ///// </summary>
    //[Newtonsoft.Json.JsonIgnore]
    //[System.Text.Json.Serialization.JsonIgnore]
    //[Navigate(NavigateType.OneToOne, nameof(CreateUserId))]
    //public virtual SysUser CreateUser { get; set; }

    /// <summary>
    /// 创建者姓名
    /// </summary>
    [SugarColumn(ColumnDescription = "创建者姓名", Length = 64, IsOnlyIgnoreUpdate = true)]
    public virtual string? CreateUserName { get; set; }

    /// <summary>
    /// 修改者Id
    /// </summary>
    [SugarColumn(ColumnDescription = "修改者Id")]
    public virtual long? UpdateUserId { get; set; }

    ///// <summary>
    ///// 修改者
    ///// </summary>
    //[Newtonsoft.Json.JsonIgnore]
    //[System.Text.Json.Serialization.JsonIgnore]
    //[Navigate(NavigateType.OneToOne, nameof(UpdateUserId))]
    //public virtual SysUser UpdateUser { get; set; }

    /// <summary>
    /// 修改者姓名
    /// </summary>
    [SugarColumn(ColumnDescription = "修改者姓名", Length = 64)]
    public virtual string? UpdateUserName { get; set; }

    /// <summary>
    /// 软删除
    /// </summary>
    [SugarColumn(ColumnDescription = "软删除")]
    public virtual bool IsDelete { get; set; } = false;
}
```

## 3、包括部门Id、部门名称
:::tip 提示

有数据权限需求时继承此基类，比如数据分部门、分上下级等。系统自动赋值当前操作者部门Id、部门名称，应用层无需手动处理。
:::
```csharp
/// <summary>
/// 业务数据实体基类（数据权限）
/// </summary>
public abstract class EntityBaseData : EntityBase, IOrgIdFilter
{
    /// <summary>
    /// 创建者部门Id
    /// </summary>
    [SugarColumn(ColumnDescription = "创建者部门Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? CreateOrgId { get; set; }

    /// <summary>
    /// 创建者部门
    /// </summary>
    [Newtonsoft.Json.JsonIgnore]
    [System.Text.Json.Serialization.JsonIgnore]
    [Navigate(NavigateType.OneToOne, nameof(CreateOrgId))]
    public virtual SysOrg CreateOrg { get; set; }

    /// <summary>
    /// 创建者部门名称
    /// </summary>
    [SugarColumn(ColumnDescription = "创建者部门名称", Length = 64, IsOnlyIgnoreUpdate = true)]
    public virtual string? CreateOrgName { get; set; }
}
```

## 4、包括租户Id及其他基础信息
:::tip 提示

有多租户需求时继承此基类。系统自动赋值当前操作者租户Id，应用层无需手动处理。
:::
```csharp
/// <summary>
/// 租户实体基类
/// </summary>
public abstract class EntityTenant : EntityBase, ITenantIdFilter
{
    /// <summary>
    /// 租户Id
    /// </summary>
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```

## 5、只包括租户Id
:::tip 提示

有多租户需求时继承此基类且只关心租户Id。系统自动赋值当前操作者租户Id，应用层无需手动处理。
:::
```csharp
/// <summary>
/// 租户实体基类Id
/// </summary>
public abstract class EntityTenantId : EntityBaseId, ITenantIdFilter
{
    /// <summary>
    /// 租户Id
    /// </summary>
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```
## 6、括租户Id、部门Id
:::tip 提示

有多租户需求时继承此基类且租户内还分数据权限。系统自动赋值当前操作者租户Id、部门Id等，应用层无需手动处理。
:::
```csharp
/// <summary>
/// 租户实体基类 + 业务数据（数据权限）
/// </summary>
public abstract class EntityTenantBaseData : EntityBaseData, ITenantIdFilter
{
    /// <summary>
    /// 租户Id
    /// </summary>
    [SugarColumn(ColumnDescription = "租户Id", IsOnlyIgnoreUpdate = true)]
    public virtual long? TenantId { get; set; }
}
```