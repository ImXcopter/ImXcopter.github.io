# HTML可以嵌入PHP“ if”语句中吗？

是的，可以在PHP的帮助下将HTML嵌入“ if”语句中。以下是一些方法。
使用if条件-

```
php if($condition) : ?
   it is displayed iff $condition is met
php endif; ?
```

使用if和else if条件-

```
php if($condition) : ?
    it is displayed iff $condition is met 
php elseif($another_condition) : ?
   HTML TAG HERE
php else : ?
   HTML TAG HERE
php endif; ?
```

在PHP内嵌入HTML-

```
php
   if ( $condition met ) {
      ? HTML TAG HERE
php;
}
?
```