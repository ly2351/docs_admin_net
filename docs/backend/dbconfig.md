---
sidebar_position: 8
---

# 8. 数据库配置


:::tip 提示

Admin.NET 默认以 codefirst 建库建表，代码优先。数据库配置文件在应用层 Configuration 文件夹下面 Database.json 配置文件里面。直接修改数据库类型和数据库连接字符串，记得重新生成解决方案让配置生效，程序启动自动生成数据库和种子数据，不需要任何 sql 脚本文件。

:::

:::tip 提示

若实体或者种子数据没有发生变化，可以在生成数据后将配置改为 false，比如 数据库初始化EnableInitDb：false，库表结构初始化EnableInitTable：false, 种子数据初始化EnableInitSeed：false。否则每次启动服务会冲掉数据库已有数据。

:::

:::tip 提示

若项目中库表和种子数据文件特别多，则开启库表和种子初始化开关后，可以只在更新的库表和种子类上面贴特性 [IncreTable]，保证只更新这个表和种子数据文件，以提升更新、启动速度。

:::

Admin.NET 默认数据库类型 Sqlite，路径为项目 Admin.NET.Web.Entry 下面 DataSource=./Admin.NET.db, 在应用层 Configuration 文件夹下面 Database.json 配置文件里面自行修改。根据业务需要可自动更换各种数据库，只需要修改数据库类型和连接字符串即可，系统自动创建数据库表，无需任何 SQL 脚本文件。

```csharp
{
  // 详细数据库配置见SqlSugar官网（第一个为默认库）
  "DbConnection": {
    "EnableConsoleSql": true, // 启用控制台打印SQL
    "ConnectionConfigs": [
      {
        //"ConfigId": "1300000000001", // 默认库标识-禁止修改
        "DbType": "Sqlite", // MySql、SqlServer、Sqlite、Oracle、PostgreSQL、Dm、Kdbndp、Oscar、MySqlConnector、Access、OpenGauss、QuestDB、HG、ClickHouse、GBase、Odbc、Custom
        "ConnectionString": "DataSource=./Admin.NET.db", // 库连接字符串
        //"SlaveConnectionConfigs": [ // 读写分离/主从
        // {
        //  "HitRate": 10,
        //  "ConnectionString": "DataSource=./Admin.NET1.db" // 从库1
        // },
        // {
        //  "HitRate": 10,
        //  "ConnectionString": "DataSource=./Admin.NET2.db" // 从库2
        // }
        //],
        "DbSettings": {
          "EnableInitDb": false, // 启用库初始化-若实体没有变化则关闭以提升启动速度
          "EnableDiffLog": false, // 启用库表差异日志
          "EnableUnderLine": false // 启用驼峰转下划线
        },
        "TableSettings": {
          "EnableInitTable": false, // 启用表初始化-若实体没有变化则关闭以提升启动速度
          "EnableIncreTable": false // 启用表增量更新-特性[IncreTable]
        },
        "SeedSettings": {
          "EnableInitSeed": false, // 启用种子初始化-若种子没有变化则关闭以提升启动速度
          "EnableIncreSeed": false // 启用种子增量更新-特性[IncreSeed]
        }
      }
    ]
  }
}
```

数据库连接字符串（其他库类型参考 https://www.donet5.com/Home/Doc）

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
<TabItem value="js" label="PostgreSQL">

```js
PORT=5432;DATABASE=adminnet;HOST=localhost;PASSWORD=123456;USER ID=postgres
```

</TabItem>
<TabItem value="py" label="SqlServer">

```py
def hello_world():
  print("Hello, world!")
```

</TabItem>
<TabItem value="java" label="MySql">

```java
class HelloWorld {
  public static void main(String args[]) {
    System.out.println("Hello, World");
  }
}
```

</TabItem>
</Tabs>

支持的数据库，但不限以下：

```csharp
- MySql
- SqlServer
- Sqlite
- Oracle
- postgresql
- 达梦
- 人大金仓
- 海量数据库Vastbase
- 神通数据库
- 瀚高
- 华为 GaussDB
- 南大通用 GBase
- TDSQL
- ......
```

:::tip 强烈推荐数据库 PostgreSQL

世界上最先进的开源关系数据库https://www.postgresql.org/ ，数据量越大越牛逼，同时具备各种数据处理插件，比如GIS插件 PostGIS、时序插件 Timescale等。
:::

