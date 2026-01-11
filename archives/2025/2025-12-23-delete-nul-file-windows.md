# nul 文件无法删除的解决方法

使用 ClaudeCode 的时候，经常文件夹中有莫名其妙的 nul 文件，右键删除不掉怎么办？

## 解决方法

1. 新建 `.txt` 文件
2. 粘贴代码：

```batch
DEL /F /A /Q \\?\%1
RD /S /Q \\?\%1
```

3. 修改文件名为 `del.bat`
4. 将 nul 文件拖入到 del.bat 文件
