# HTML可以嵌入PHP"if"语句中吗？

是的，可以在 PHP 的帮助下将 HTML 嵌入"if"语句中。以下是一些方法。

### 使用if条件

```text
php if($condition) : ?
   it is displayed iff $condition is met
php endif; ?
```

### 使用if和else if条件

```text
php if($condition) : ?
    it is displayed iff $condition is met
php elseif($another_condition) : ?
   HTML TAG HERE
php else : ?
   HTML TAG HERE
php endif; ?
```

### 在PHP内嵌入HTML

```text
php
   if ( $condition met ) {
      ? HTML TAG HERE
php;
}
?
```
