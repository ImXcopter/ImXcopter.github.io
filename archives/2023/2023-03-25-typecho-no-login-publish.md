# Typecho 免登陆接口文章发布模块

Typecho 火车头免登陆接口文章发布模块，适用于 Typecho 1.1 版本。

## 功能简介

1. 应用于 Typecho 1.1 文章发布
2. 支持多用户账号发布文章，账号应具备发布权限
3. 接口请上传在网站目录下使用

## 配置说明

配置 `Conn.php` 文件，按照说明修改：

```php
<?php

$password = '123456';  // 这个密码是登陆验证用的。您需要在模块里设置和这里一样的密码....注意一定需要修改。

$db_config = array (
    'host' => 'localhost',    // 数据库地址
    'user' => 'user',         // 数据库用户名
    'password' => '123456',   // 数据库密码
    'charset' => 'utf8',      // 数据库字符集
    'port' => '3306',         // 数据库端口
    'database' => 'typecho',  // 数据库名称
    'tablepre' => 'typecho_', // 数据表前缀
);

?>
```

## 使用说明

如果修改了接口验证密码，请打开发布模块修改，并将接口文件上传到网站的目录下。

## 下载地址

[Typecho 免登陆接口.zip](https://attachment.xcopter.cc/2023/Typecho%E5%85%8D%E7%99%BB%E9%99%86%E6%8E%A5%E5%8F%A3.zip)
