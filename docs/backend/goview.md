---
sidebar_position: 24
---

# 24. 可视化大屏

:::tip 提示

Admin.NET 内置了可视化大屏接口服务。[GoView](https://gitee.com/dromara/go-view) 是一个 Vue3 搭建的低代码数据可视化开发平台，将图表或页面元素封装为基础组件，无需编写代码即可完成业务需求。开源、精美、便捷的「数据可视化」低代码开发平台。[GoView 教程](https://www.mtruning.club/)
:::

Admin.NET 实现 GoView 可视化大屏接口服务是以 `插件` 模式提供的，工程名字 `Admin.NET.Plugin.GoView`，需要的只需要将此工程添加引用到自己的应用工程里面即可。也提倡大家将平时共用的业务功能单独创建一个插件工程去实现，达到解耦共用目的。

GoView 前端工程代码请自行下载，记得下载带后端请求分支 [master-fetch](https://gitee.com/dromara/go-view/tree/master-fetch/)。

修改 GoView 接口服务地址 [master-fetch/.env](https://gitee.com/dromara/go-view/blob/master-fetch/.env)。

```env
# port
VITE_DEV_PORT = '8080'

# development path
VITE_DEV_PATH = 'http://localhost:5005'

# production path
VITE_PRO_PATH = 'http://localhost:5005'
```

公共前缀修改：`src\settings\httpSetting.ts`

```ts
# port
// 请求前缀
export const axiosPre = '/api/goview'
```

依次运行命令`pnpm install`、`pnpm run dev`启动工程即可。此时GoView登录、大屏项目的增删改查都和自己的业务库相关联了。无需写一行代码 😎，就可以自由设计大屏。

接口封装：`src\api\http.ts`

```ts
import axiosInstance from './axios'
import { RequestHttpEnum, ContentTypeEnum } from '@/enums/httpEnum'

export const get = (url: string, params?: object) => {
  return axiosInstance({
    url: url,
    method: RequestHttpEnum.GET,
    params: params,
  })
}

export const post = (url: string, data?: object, headersType?: string) => {
  return axiosInstance({
    url: url,
    method: RequestHttpEnum.POST,
    data: data,
    headers: {
      'Content-Type': headersType || ContentTypeEnum.JSON
    }
  })
}

export const put = (url: string, data?: object, headersType?: string) => {
  return axiosInstance({
    url: url,
    method: RequestHttpEnum.PUT,
    data: data,
    headers: {
      'Content-Type': headersType || ContentTypeEnum.JSON
    }
  })
}

export const del = (url: string, params?: object) => {
  return axiosInstance({
    url: url,
    method: RequestHttpEnum.DELETE,
    params
  })
}

// 获取请求函数，默认get
export const http = (type?: RequestHttpEnum) => {
  switch (type) {
    case RequestHttpEnum.GET:
      return get

    case RequestHttpEnum.POST:
      return post

    case RequestHttpEnum.PUT:
      return put

    case RequestHttpEnum.DELETE:
      return del

    default:
      return get
  }
}
```

GoView 接口说明地址 [接口地址](https://docs.apipost.cn/preview/5aa85d10a59d66ce/ddb813732007ad2b?target_id=dd81da11-9f8c-48ce-a4e8-3647279683fe)

运行效果：


![GoView 登录](./../../static/img/backend/3.png)

![创建项目](./../../static/img/backend/4.jpg)

