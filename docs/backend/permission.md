---
sidebar_position: 9
---

# 9. 数据权限


:::tip 提示

Admin.NET 采用常见的RBAC权限管理模型，给角色分配菜单、按钮及接口资源权限，然后给用户指定相应角色，支持多角色模式。有数据行权限需求时，该实体/表必须继承此基类，比如数据分部门、分上下级等。系统自动赋值当前操作者部门Id、部门名称，应用层无需手动处理。
:::

只需要正确定义实体，就可以实现数据权限过滤，就这么简单!

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

针对继承该基类的实体，系统会自动生成全局查询拦截，查询的时候自动将当前用户所属部门Id带上，进行组织架构集合过滤查询。每个用户的组织架构集合和用户所处部门级别有关，比如上级、下级。超管不受任何权限控制。

具体调用实在数据库AOP初始化时 SqlSugarFilter.SetOrgEntityFilter(db); ，这样全局查询时会自动带上部门Id进行过滤。具体实现类 SqlSugarFilter.cs

```csharp
    /// <summary>
    /// 配置用户机构集合过滤器
    /// </summary>
    public static void SetOrgEntityFilter(SqlSugarScopeProvider db)
    {
        // 若仅本人数据，则直接返回
        if (SetDataScopeFilter(db) == (int)DataScopeEnum.Self) return;

        var userId = App.User?.FindFirst(ClaimConst.UserId)?.Value;
        if (string.IsNullOrWhiteSpace(userId)) return;

        // 配置用户机构集合缓存
        var cacheKey = $"db:{db.CurrentConnectionConfig.ConfigId}:orgList:{userId}";
        var orgFilter = _cache.Get<ConcurrentDictionary<Type, LambdaExpression>>(cacheKey);
        if (orgFilter == null)
        {
            // 获取用户所属机构，保证同一作用域
            var orgIds = new List<long>();
            Scoped.Create((factory, scope) =>
            {
                var services = scope.ServiceProvider;
                orgIds = services.GetService<SysOrgService>().GetUserOrgIdList().GetAwaiter().GetResult();
            });
            if (orgIds == null || orgIds.Count == 0) return;

            // 获取业务实体数据表
            var entityTypes = App.EffectiveTypes.Where(u => !u.IsInterface && !u.IsAbstract && u.IsClass
                && u.IsSubclassOf(typeof(EntityBaseData)));
            if (!entityTypes.Any()) return;

            orgFilter = new ConcurrentDictionary<Type, LambdaExpression>();
            foreach (var entityType in entityTypes)
            {
                // 排除非当前数据库实体
                var tAtt = entityType.GetCustomAttribute<TenantAttribute>();
                if ((tAtt != null && db.CurrentConnectionConfig.ConfigId.ToString() != tAtt.configId.ToString()))
                    continue;

                var lambda = DynamicExpressionParser.ParseLambda(new[] {
                    Expression.Parameter(entityType, "u") }, typeof(bool), $"@0.Contains(u.{nameof(EntityBaseData.CreateOrgId)}??{default(long)})", orgIds);
                db.QueryFilter.AddTableFilter(entityType, lambda);
                orgFilter.TryAdd(entityType, lambda);
            }
            _cache.Add(cacheKey, orgFilter);
        }
        else
        {
            foreach (var filter in orgFilter)
                db.QueryFilter.AddTableFilter(filter.Key, filter.Value);
        }
    }
```

**支持自定义过滤条件** 比如定义某实体不仅按照部门Id进行过滤，还可以自行定义过滤条件，比如按照名称、编码进行过滤等。具体实现类 SqlSugarFilter.cs

```csharp
    /// <summary>
    /// 配置自定义过滤器
    /// </summary>
    public static void SetCustomEntityFilter(SqlSugarScopeProvider db)
    {
        // 配置自定义缓存
        var userId = App.User?.FindFirst(ClaimConst.UserId)?.Value;
        var cacheKey = $"db:{db.CurrentConnectionConfig.ConfigId}:custom:{userId}";
        var tableFilterItemList = _cache.Get<List<TableFilterItem<object>>>(cacheKey);
        if (tableFilterItemList == null)
        {
            // 获取自定义实体过滤器
            var entityFilterTypes = App.EffectiveTypes.Where(u => !u.IsInterface && !u.IsAbstract && u.IsClass
                && u.GetInterfaces().Any(i => i.HasImplementedRawGeneric(typeof(IEntityFilter))));
            if (!entityFilterTypes.Any()) return;

            var tableFilterItems = new List<TableFilterItem<object>>();
            foreach (var entityFilter in entityFilterTypes)
            {
                var instance = Activator.CreateInstance(entityFilter);
                var entityFilterMethod = entityFilter.GetMethod("AddEntityFilter");
                var entityFilters = ((IList)entityFilterMethod?.Invoke(instance, null))?.Cast<object>();
                if (entityFilters == null) continue;

                foreach (var u in entityFilters)
                {
                    var tableFilterItem = (TableFilterItem<object>)u;
                    var entityType = tableFilterItem.GetType().GetProperty("type", BindingFlags.Instance | BindingFlags.NonPublic).GetValue(tableFilterItem, null) as Type;
                    // 排除非当前数据库实体
                    var tAtt = entityType.GetCustomAttribute<TenantAttribute>();
                    if ((tAtt != null && db.CurrentConnectionConfig.ConfigId.ToString() != tAtt.configId.ToString()) ||
                        (tAtt == null && db.CurrentConnectionConfig.ConfigId.ToString() != SqlSugarConst.MainConfigId))
                        continue;

                    tableFilterItems.Add(tableFilterItem);
                    db.QueryFilter.Add(tableFilterItem);
                }
            }
            _cache.Add(cacheKey, tableFilterItems);
        }
        else
        {
            tableFilterItemList.ForEach(u =>
            {
                db.QueryFilter.Add(u);
            });
        }
    }
```

应用层只需要实现继承数据权限接口 `IEntityFilter` 即可，示例：

```csharp
/// <summary>
/// 自定义业务实体过滤器示例
/// </summary>
public class TestEntityFilter : IEntityFilter
{
    public IEnumerable<TableFilterItem<object>> AddEntityFilter()
    {
        // 构造自定义条件的过滤器
        Expression<Func<SysUser, bool>> dynamicExpression = u => u.Remark.Contains("xxx");
        var tableFilterItem = new TableFilterItem<object>(typeof(SysUser), dynamicExpression);

        return new[]
        {
            tableFilterItem
        };
    }
}
```