---
date: 2018-03-27 16:43:36
tags:
- Web 开发
- Asp.Net
- Vue
title: 在 ABP 1.1 和 Vue 单页应用中的踩坑总结
---

# 前言

离 <http://acm.nwpu.edu.cn> 开发结束已经过去半年多了，我最近也放弃了考研的打算，现在也算是有空写博客了\_(:3」∠)\_。我就在这一篇博文中总结一下之前踩过的坑。注意，这篇博文中的后端技术仅限于 Asp.Net Core 1.1 及 AspNetBoilerplate 2.3.0。我不保证这里的解决方案在现在的 .net core 2.0 和 ABP 3.X.X 里还能一样工作。另外，直接写在ABP、VUE文档里或者用Google能直接查到的解决方案我也是懒得在这里再写一遍的。

这个网站是一个前后端完全分离的应用，后端用 ABP 来提供 api 供前端调用；前端是用 vue 编写的单页应用，在构建过程中会直接生成出目标 html 和 js，放进 .net core 的 `wwwroot` 中。在部署环境中由后端来提供前端的静态文件。

# 后端ABP及ASP.NET CORE相关
## 把 Service 的注释文档放进 Swagger UI 中
默认情况下 Abp 生成的 swagger 里面只有Service 的方法名，没有文档。我们可以把Service 方法里 `summary` tag的里面的注释输出成 swagger 的文档。效果是这样的：
![](1.png)
会输出成：
![](2.png)

### 第一步：输出文档
直接修改业务层 `XXX.Application` 的属性，生成 XML 文档：
![](3.png)

### 第二步：展示文档
修改 `XXX.Web.Host/Startup/Startup.cs` 文件，加上两行代码：
```csharp
using System.IO;
using Microsoft.Extensions.PlatformAbstractions;
```
找到同一个文件的 `services.AddSwaggerGen` 那一行，在配置函数里加上
```csharp
#if (!DEBUG)
//Set the comments path for the swagger json and ui.
var basePath = PlatformServices.Default.Application.ApplicationBasePath;
var docPath = Path.Combine(basePath, "XXX.Application.xml");
options.IncludeXmlComments(docPath);
#endif
```
这里的 docPath 是生成的文档文件的目录。我的设定是在 production 模式下生成文档，所以这里直接定位了build出来的文档的位置。

## 将 authorization token 放进 cookie 中
一开始我们的后端API是遵照 rest 标准的

# 持续集成
## 用 travis-ci 构建项目
