---
layout: post
title: 使用jeklly搭建自己的blog
tag: jekyll
---


安装
安装jekyll非常容易 安装ruby后，使用gem一行命令搞定

```ruby
gem install jekyll
```

具体参看:

[https://github.com/mojombo/jekyll/wiki/install](https://github.com/mojombo/jekyll/wiki/install)

<!--more-->

使用
使用起来很简单。按照 jekyll的标准设置目录，修改_config.yml配置文件，然后用



`jekyll --server`


###扩展解决方案
默认jekyll的首页只展示文章列表。可以让显示全文，但会导致首页过大。所以希望通过摘要的方式实现。博客摘要的功能当前有几种实现方式：

1. 插件方式

插件方式功能较强，但github的pages服务不支持插件。

_plugins/more.rb:

```ruby
 module More
     def more(input, type)
         if input.include? "<!--more-->"
             if type == "excerpt"
                 input.split("<!--more-->").first
             elsif type == "remaining"
                 input.split("<!--more-->").last
             else
                 input
             end
         else
             input
         end
     end
 end

 Liquid::Template.register_filter(More)
 ```


2.  博客中增加标记

```html
 ---
 layout: post
 title: "Your post title"
 published: true
 ---
 <p>This is the excerpt.</p>
 <!--more-->
 <p>This is the remainder of the post.</p>

```

获取摘要方法

```html
 <summary></summary>
```


3.  liquid 过滤器方式

这种模式不需要插件，但缺点是截取的字符是固定的，会把一个词切开，体验不好。


通过html的注解进行hacks

在文档中增加特殊的标记。


```html
 ---
 title: some post
 layout: post
 ---
 Some intro, this will be visible on the index page.
 <!-- more start -->
 More content, this will not be visible on the index page.
 <!-- more end -->

```
显示summary的时候通过文本替换，将后面的全文内容放到html的注释中。这样页面上就不会显示出来。虽然看起来会好一点，但白白传输了一部分不需要显示的内容。

来源: http://kaspa.rs/2011/04/jekyll-hacks-html-excerpts/



1. js操作模式

思路是页面渲染后通过js把过长的文章中的html节点给隐藏了。 具体参看: http://blog.evercoding.net/2013/03/09/traversing-dom-tree-with-javascript/



1. 我的解决方案

参考方案3，我的解决方案更简单:

```html
 ---
 title: some post
 layout: post
 ---
 Some intro, this will be visible on the index page.
 <!--more-->
 More content, this will not be visible on the index page.
```

然后用split方法截取第一部分显示

```html
 post.content | split:'<!--more-->' |first
```

1. 代码高亮

代码高亮有两种解决方案

使用pygmentize或者redcarpet在生成页面的时候自动将代码格式化成带样式的html 参考: https://github.com/mojombo/jekyll/wiki/install

纯js解决方案

有名的有,google-code-prettify和 SyntaxHighlighter。后者功能比较强大，但也比较臃肿,最后选择了 google-code-prettify

```html
 <script type="text/javascript" src="/js/jquery-1.9.1.min.js"></script>
 <script type="text/javascript">
     $(function() {
         $('pre').addClass('prettyprint').attr('style', 'overflow:auto');
     });
     $(function() {
          window.prettyPrint && prettyPrint();
     });
 </script>
 <script src="/google-code-prettify/prettify.js" defer="defer"></script>
```



参考资料:

[http://www.zhanxin.info/jekyll/2013-08-07-jekyll-configuration.html](http://www.zhanxin.info/jekyll/2013-08-07-jekyll-configuration.html)

[http://jolestar.com/use-jekyll-as-blog/](http://jolestar.com/use-jekyll-as-blog/)




