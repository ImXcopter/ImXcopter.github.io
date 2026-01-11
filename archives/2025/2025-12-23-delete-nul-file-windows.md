# nul文件无法删除的解决方法

使用ClaudeCode的时候，经常文件夹中有莫名其妙的nul文件，右键删除不掉怎么办？
1.新建.txt文件
2.粘贴代码：

```
DEL /F /A /Q \\?\%1
RD /S /Q \\?\%1
```

3.修改文件名为del.bat
4.将nul文件拖入到del.bat文件