---
sidebar_position: 14
---

# 14. 图片压缩

:::tip 提示

同一张图片，前端根据不同应用场景自行设置图片格式、大小、旋转、水印等。

SixLabors.ImageSharp 是一个强大的、开源的、跨平台的.NET库，用于处理和操作各种图像文件。它提供了一套完整的API，允许开发者在他们的应用中进行实时的图像处理，如缩放、旋转、裁剪，以及其他复杂的滤镜效果。
:::

### 技术解析
高性能 ImageSharp 使用高度优化的代码和原生内存管理，确保了即使在大量图像处理任务下也能保持高效的速度。它的设计目标是在不牺牲质量的前提下，尽可能减少CPU和内存的使用。

跨平台兼容性 作为一个.NET库，ImageSharp 兼容多种.NET框架，包括 .NET Core 和 .NET Framework，这使得它可以无缝地在Windows、Linux和MacOS等不同操作系统上运行。

支持多种图像格式 ImageSharp 支持常见的图像格式，如JPEG、PNG、GIF、BMP和WebP，还有计划增加对更多格式的支持。不仅如此，它还具有解码和编码的能力，让你可以轻松读取和保存图像。

响应式设计 ImageSharp 提供了响应式的图像处理能力，可以根据设备屏幕尺寸和分辨率动态调整图像大小，这对于移动应用和现代Web开发尤其重要。

图像处理API 其API设计直观且易于使用，通过链式调用来实现各种图像操作，如image.Mutate(x => x.Resize(new Size(800, 600))).Save("output.jpg")，这使得即便是初学者也能够快速上手。

### 应用场景
Web 开发： 对于需要在服务器端生成或调整图片的Web应用，ImageSharp 可以极大地提高性能。

桌面应用： 无论是照片编辑工具还是数据可视化软件，ImageSharp 都能提供强大的图像处理功能。

移动应用： 在资源有限的移动设备上，ImageSharp 的轻量级和高性能使其成为理想的选择。

自动化工具： 在批量处理图像的脚本或服务中，ImageSharp 可以节省大量的时间和计算资源。

### Web 应用
Imagesharp.Web 用于在Web应用程序中处理和操作图像。它提供了一组功能强大的API，可以轻松地对图像进行裁剪、调整大小、旋转、添加水印等操作。自定义网址模式是Imagesharp.Web的一个特性，它允许开发人员根据自己的需求定义图像的URL路径。通过自定义网址模式，开发人员可以更灵活地管理和控制图像的访问方式。

自定义网址模式的优势在于：

灵活性： 开发人员可以根据自己的需求定义图像的URL路径，可以根据不同的场景和要求进行定制。

安全性： 通过自定义网址模式，可以实现对图像的访问权限控制，确保只有授权用户可以访问特定的图像。

可维护性： 自定义网址模式可以使URL路径更加直观和易于理解，提高代码的可读性和可维护性。

Imagesharp.Web的自定义网址模式可以应用于各种场景，例如：

图片存储和管理： 可以根据自定义网址模式将图像存储在不同的目录结构中，方便管理和组织。

图片处理和展示： 可以根据自定义网址模式对图像进行不同的处理和展示，例如裁剪、缩放、添加水印等。

图片访问控制： 可以通过自定义网址模式实现对图像的访问权限控制，例如只允许特定用户或特定IP地址访问。

下面是使用示例 ImageSharp.Web 教程：

### Resize 调整

```bash
{PATH_TO_YOUR_IMAGE}?width=300
{PATH_TO_YOUR_IMAGE}?width=300&height=120&rxy=0.37,0.78
{PATH_TO_YOUR_IMAGE}?width=50&height=50&rsampler=nearest&rmode=stretch
{PATH_TO_YOUR_IMAGE}?width=300&compand=true&orient=false
```

1、width 图像的宽度（以 px 为单位）

2、height 图像的高度（以 px 为单位）

3、rmode 要使用的 ResizeMode

4、rsampler 要使用的 IResampler 采样器（具体采样方法见官网教程）

5、ranchor 要使用的 AnchorPositionMode

6、rxy 使用精确的锚点位置点。逗号分隔的 x 和 y 值的范围为 0-1。

7、 orient 是否根据指示旋转（未翻转）图像的 EXIF 元数据的存在来交换命令维度。默认值为 true

8、 compand 处理时是否将单个像素颜色值压缩和扩展到线性颜色空间或从线性颜色空间扩展。默认值为 false

### Format 格式

```bash
{PATH_TO_YOUR_IMAGE}?format=bmp
{PATH_TO_YOUR_IMAGE}?format=gif
{PATH_TO_YOUR_IMAGE}?format=jpg
{PATH_TO_YOUR_IMAGE}?format=pbm
{PATH_TO_YOUR_IMAGE}?format=png
{PATH_TO_YOUR_IMAGE}?format=tga
{PATH_TO_YOUR_IMAGE}?format=tiff
{PATH_TO_YOUR_IMAGE}?format=webp
```

### Quality 质量

```bash
{PATH_TO_YOUR_IMAGE}?quality=90
{PATH_TO_YOUR_IMAGE}?format=jpg&quality=42
```

:::tip 提示

允许以给定质量对输出图像进行编码，对于 Jpeg，此范围为 1-100，对于 WebP，此范围为 1-100。只有某些格式支持可调节的质量。这是单个图像标准的约束，而不是 API 的约束。
:::

### Background Color 背景颜色

```bash
{PATH_TO_YOUR_IMAGE}?bgcolor=FFFF00
{PATH_TO_YOUR_IMAGE}?bgcolor=C1FF0080
{PATH_TO_YOUR_IMAGE}?bgcolor=red
{PATH_TO_YOUR_IMAGE}?bgcolor=128,64,32
{PATH_TO_YOUR_IMAGE}?bgcolor=128,64,32,16
```