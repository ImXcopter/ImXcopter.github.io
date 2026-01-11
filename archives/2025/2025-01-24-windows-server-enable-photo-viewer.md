# Windows Server 2008 2012 2016 2019 2022 服务器系统设置使用图片查看器

因为 windows server 服务器系统默认使用的是画图工具打开，无论是查看图片和打印图片等，有时候都是很不方便的，下面提供解决方法。

## 方法步骤

1. 直接按快捷键：win 键 + R，打开：运行，输入：`regedit`，敲回车键，打开：注册表编辑器。

2. 依次展开注册表路径：

```reg
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows Photo Viewer\Capabilities\FileAssociations
```

3. 在右侧新建字符串项：`.jpg` 和 `.png`，数据值为：`PhotoViewer.FileAssoc.Tiff`

![](/static/2025/2025-01-24-windows-server-enable-photo-viewer_001.png)
