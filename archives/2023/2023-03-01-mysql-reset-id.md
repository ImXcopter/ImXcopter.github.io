# MySQL ID 重新排列方法

我们使用 MySQL 经常会遇到，在删除数据库某条记录时，原来的 ID 排序会有间隔，比如删除了 ID 为 8 的数据，这个表的 ID 排序就会从 7 直接到 9，那我们如何解决这个 ID 重新排列的问题呢？

只需以下三步：

**1. 删除这个表的 ID**

```sql
ALTER TABLE `table_name` DROP `id`;
```

**2. 重新建立 ID 字段**

```sql
ALTER TABLE `table_name` ADD `id` MEDIUMINT(8) NOT NULL FIRST;
```

**3. 为这个字段设置新的主键，并且自动增长**

```sql
ALTER TABLE `table_name` MODIFY COLUMN `id` MEDIUMINT(8) NOT NULL AUTO_INCREMENT, ADD PRIMARY KEY(id);
```
