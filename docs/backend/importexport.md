---
sidebar_position: 5
---

# 5. 导入导出

:::tip 提示

Admin.NET 导入导出采用组件 Magicodes.IE，支持Excle、Word、PDF、Html等格式。若要以 Word 模板导出，推荐组件 MiniWord，系统计划集成此组件。
:::

1、首先定义导入导出 DTO 映射文件，导入 ImporterHeader 和导出 ExporterHeader，如下：

```csharp
/// <summary>
/// xxx
/// </summary>
[ExcelImporter(IsLabelingError = true)]
public class xxxDto
{
    /// <summary>
    /// 名称
    /// </summary>
    [ImporterHeader(Name = "名称")]
    public string Name { get; set; }

    /// <summary>
    /// 金额
    /// </summary>
    [ImporterHeader(Name = "金额")]
    public decimal Amount { get; set; }
}
```

2、后台读取文件并解析，如下：

```csharp
public class xxxService : IDynamicApiController, ITransient
{
    public OrderBillService()
    {

    }

    // 按模板导入
    public async Task<dynamic> Importxxx([Required] IFormFile file, [Required] long supplierId)
    {
        var newFile = await App.GetRequiredService<SysFileService>().UploadFile(file, "");
        var filePath = Path.Combine(App.WebHostEnvironment.WebRootPath, newFile.FilePath, newFile.Id.ToString() + newFile.Suffix);

        IImporter importer = new ExcelImporter();
        var res = await importer.Import<xxxDto>(filePath);
        if (res == null || res.Exception != null)
            throw Oops.Oh("导入异常:" + res.Exception);
        
        // 此时取到 res.Data 就是外面导入的表格数据，自行处理。。。
    }

    // 按模板导出
    public async Task<IActionResult> Exportxxx()
    {
        IImporter importer = new ExcelImporter();
        var tempPath = AppContext.BaseDirectory + "temp\\xxx.xlsx";
        var fileName = DateTime.Now.ToString("yyyyMMdd") + Path.GetFileName(tempPath);
        var res = await importer.GenerateTemplate<xxxDto>(App.WebHostEnvironment.WebRootPath + "\\" + fileName);
        return new FileStreamResult(new FileStream(res.FileName, FileMode.Open), "application/octet-stream") { FileDownloadName = fileName };
    }
}
```

**前端上传与下载请参考** [上传下载](uploaddownload.md)