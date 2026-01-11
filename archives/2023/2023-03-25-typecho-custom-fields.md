# Typecho教程 - 自定义字段相关

一、获取模板自定义字段值
------------

在 Typecho 很多模板都要通过设置自定义字段来实现文章缩略图或者其他功能，但是我们在二次开发或者开发插件时，并没有一个接口来实现获取自定义字段，所以便有了我今天的想法。
**代码**

```
public function getCustom($cid, $key){
    $db = Typecho_Db::get();
    $rows = $db->fetchAll($db->select('table.fields.str_value')->from('table.fields')
        ->where('table.fields.cid = ?', $cid)
        ->where('table.fields.name = ?', $key)
    );
    // 如果有多个值则存入数组
    foreach ($rows as $row) {
        $img = $row['str_value'];
        if (!empty($img)) {
            $values[] = $img;
        }
    }
    return $values;
}
```

**使用**
使用时只要使用 $this->getCustom(mix $cid, mix $key) 就可以了，两个参数分别是文章 cid 和自定义字段名，函数会把自定义内容返回成数组。

二、typecho自定义字段使用说明
------------------

通过自定义字段我们可以让我们的文章页模板如虎添翼。让typeho博客变得强大，可以是相册、可以是下载站、可以是小说。
**typecho自定义字段完全**
kavico替换为你的自定义字段

```
php if (isset($this-fields->kavico)): ?>
php $this-fields->kavico(); ?>
php else: ?
没有设置说明
php endif; ?
```

**代码说明**

```
php if (isset($this-fields->kavico)): ?>//如果有kavico这个自定义字段
php $this-fields->kavico(); ?>//输出自定义字段内容
php else: ?//如果没有kavico这个自定义字段
没有设置说明//如果没有设置时默认显示的内容
php endif; ?//判断结束
```

**直接输出自定义字段内容**
kavico 替换为你的自定义字段

```
php if (isset($this-fields->kavico)): ?>
php $this-fields->kavico(); ?>
php endif; ?
```

**通过自定义字段输出幻灯片**
'pageSize=6&type=tag', 'slug=kavico'
表示输出6条，标签缩略名为kavico的内容。（幻灯片需要自行整合焦点图、幻灯片jQuery）
文字内容中自定义字段名为：banner

```
php $this-widget('Widget_Archive@indexfocus', 'pageSize=6&type=tag', 'slug=kavico')->to($indexfocus); ?>php while($indexfocus-next()): ?>* ![<?php $indexfocus->title() ?>](<?php if (isset($indexfocus->fields->banner)): ?><?php $indexfocus->fields->banner(); ?><?php endif; ?>)
php endwhile; ?
```

三、Typecho自定义字段并将其集成在主题中
-----------------------

Typecho在主题模板functions.php里面添加下面1.代码，你就会发现你在Typecho后台撰写新文章时候下面自定义字段就会有相关的输入框了，ps:里面的ico自定义你喜欢的，但是这个ico你改了，后面的引用地方你也要改相对应的，比如有ico的字符的地方，还有里面的中文提示也可以改，这个中文提示就随你了。
1.在主题里面添加函数

```
// 文章页自定义字段
function themeFields($layout) {
$url = new Typecho_Widget_Helper_Form_Element_Text('url', NULL, NULL, _t('链接地址'), _t('在这里填入网站地址'));
$layout->addItem($url);
$ico = new Typecho_Widget_Helper_Form_Element_Text('ico', NULL, NULL, _t('自定义站标'), _t('xx/favicon.ico或/usr/themes/flkc/img/faviconX.png'));
$layout->addItem($ico);
}
```

2.引用Typecho文章自定义字段

```
//第一条引用例子
php echo $post['fields']['url'];?
```

```
//第二条引用例子
php echo $post['fields']['ico'];?
```

3.在主题里面实现引用例子

```
 php if ( !empty($post['fields']['ico']) ) :? ![](<?php echo $post['fields']['ico'];?>) php else: ? ![](<?php echo $post['fields']['url'];?>/favicon.ico)php endif;?
```

4.代码说明

```
php if ( !empty($post['fields']['ico']) ) :?//如果有ico这个自定义字段2框
![](<?php echo $post['fields']['ico'];?>)//就输出ico自定义字段的内容2框
php else: ? ![](<?php echo $post['fields']['url'];?>/favicon.ico)//如果没有侧设置时默认显示的内容1框
php endif;?//判断结束
```

5.详细说明
然后你随便在自定义字段输入框里面输入什么你想显示的文字，它就显示了！
但是你在第一个框“链接地址”写入了自定义文字比如aaa，第二个框“自定义站标”不写入自定义文字的话，上面的那个两条例子就会输出你写的aaa！
反之你在第二个框“自定义站标”写入bbb，上面的那个两条例子3.就会输出你写的bbb，不会输出aaa！因为里面判断了是不是存在ico，存在的话就输出ico的自定义字段！
我是拿来做一个默认图片和自定义图片输出的！