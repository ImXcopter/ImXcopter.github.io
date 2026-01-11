# HTML 可以嵌入 PHP "if" 语句中吗？

是的，可以在 PHP 的帮助下将 HTML 嵌入 "if" 语句中。以下是一些方法。

## 使用 if 条件

```php
<?php if($condition): ?>
   it is displayed iff $condition is met
<?php endif; ?>
```

## 使用 if 和 else if 条件

```php
<?php if($condition): ?>
    it is displayed iff $condition is met
<?php elseif($another_condition): ?>
    HTML TAG HERE
<?php else: ?>
    HTML TAG HERE
<?php endif; ?>
```

## 在 PHP 内嵌入 HTML

```php
<?php
   if ( $condition met ) {
      ?>
      HTML TAG HERE
      <?php
   }
?>
```
