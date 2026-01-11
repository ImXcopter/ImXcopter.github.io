# 使用 Chrome 命令行将HTML生成PDF

在日常工作中，我们经常需要将 HTML 文件转换为 PDF 格式。Chrome 浏览器不仅是一个强大的网页浏览工具，还是一个优秀的 PDF 生成器。通过 Chrome 的命令行接口，我们可以实现自动化 PDF 生成，并且可以灵活控制 PDF 的输出格式。本文将深入探讨如何使用 Chrome 命令行生成无页眉和页脚的 PDF 文件，让你的 PDF 更加干净整洁。

为什么选择 Chrome 命令行？
-----------------

自动化: 可以通过批处理脚本或自动化工具批量生成 PDF，提高工作效率。
可定制: 可以灵活控制 PDF 的输出格式，如禁用页眉页脚、指定纸张大小、设置缩放比例等。
跨平台: Chrome 命令行在 Windows、macOS 和 Linux 上都可以使用。
无需额外工具: 只要安装了 Chrome 浏览器，就可以使用其命令行功能。

准备工作:
-----

安装 Chrome 浏览器: 确保你的电脑上已经安装了 Chrome 浏览器。
了解 Chrome 安装路径: 你需要知道 Chrome 可执行文件 chrome.exe (Windows) 或 Google Chrome (macOS/Linux) 的安装路径。
Windows: 通常在 C:\Program Files\Google\Chrome\Application 或 C:\Program Files (x86)\Google\Chrome\Application。
macOS: 通常在 /Applications/Google Chrome.app/Contents/MacOS/Google Chrome。
Linux: 通常在 /usr/bin/google-chrome 或 /opt/google/chrome/chrome。
准备 HTML 文件: 你需要一个 HTML 文件，用于生成 PDF。
核心命令:
以下是生成无页眉页脚 PDF 的核心命令：

```
"C:\Program Files\Google\Chrome\Application\chrome.exe" --headless --print-to-pdf="C:\output.pdf" --disable-gpu --no-pdf-header-footer --print-to-pdf-no-header "file:///C:/output.html" > nul
```

命令参数解释:
-------

"C:\Program Files\Google\Chrome\Application\chrome.exe": Chrome 可执行文件的完整路径。请根据你的实际安装路径修改。
--headless: 以无头模式运行 Chrome，即不显示浏览器窗口。
--print-to-pdf="C:\output.pdf": 指定生成的 PDF 文件路径。请根据你的实际需求修改。
--disable-gpu: 禁用 GPU 加速，有时可以解决一些渲染问题。
--no-pdf-header-footer: 关键参数，禁用所有 PDF 页眉和页脚。
--print-to-pdf-no-header: 移除默认的页眉。
"file:///C:/output.html": 指定要转换为 PDF 的 HTML 文件路径。请根据你的实际需求修改，注意使用 file:/// 协议。
> nul: 将命令的标准输出重定向到空设备，屏蔽输出信息。

如何使用:
-----

打开命令行终端 (CMD 或 PowerShell): 在 Windows 上打开 CMD 或 PowerShell，在 macOS 和 Linux 上打开终端。
复制粘贴命令: 将上面的命令复制粘贴到命令行终端中，并根据你的实际路径修改。
执行命令: 按下回车键执行命令。
查看 PDF: 生成的 PDF 文件将保存在你指定的路径。

进阶技巧:
-----

创建批处理文件 (.bat 或 .cmd): 将命令保存到 .bat 或 .cmd 文件中，双击即可运行。
添加其他参数: Chrome 命令行提供了许多其他参数，你可以根据需求进行调整。例如：
--paper-size=A4: 指定纸张大小。
--print-background: 打印背景颜色和图片。
--scale=0.8: 设置缩放比例。
--margin-top=10 --margin-bottom=10 --margin-left=10 --margin-right=10: 设置页边距。