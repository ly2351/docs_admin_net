---
sidebar_position: 7
---

# 7. 模板打印


:::tip 提示

模板打印采用组件 vue-plugin-hiprint，是一个 web 打印的组件，可视化拖拽，可视化配置，手动函数添加，打印设计、可视化设计器、报表设计、元素编辑、可视化打印编辑，打印操作简单，运行快速。预览界面为css+html。支持数据分组、批量预览，生产pdf、图片更方便。
:::

依次打开菜单【平台管理】-【打印模板】，根据自己的业务应用需求新建一个模板进行保存，数据库里面存储是模板的JSON内容，设定相应的模板名称，如下图：

![模板设置](../../static/img/backend/1.jpg)

根据数据库里面已有的打印模板，前端指定模板名称进行获取模板 JSON 内容，进行打印，支持批量，打印的内容就是 JSON，JSON 关键字和模板定义的时候保持一致即可。

```csharp
const print = () => {
 formRef.value.validate(async (valid: boolean) => {
  if (!valid)  return;

  var template = JSON.parse(state.printFormData.template); // 获取指定的打印模板
   if (template == undefined || template == null) {
    ElMessage.error('打印模板错误');
    return;
   }

   let hiprintTemplate = new hiprint.PrintTemplate({
    template: template,
   });

   
   hiprintTemplate.print({ xxx1: "", xxx2: "" }); // 单个打印

   hiprintTemplate.print(JSON集合); // 批量打印

 });
};
```

:::tip 提示

从后台数据库里面取出来的模板内容一定要JSON格式化 JSON.parse()，否则打印模板无法识别。
:::