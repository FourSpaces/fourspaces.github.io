---
layout: post
title: Bootstrap 附加导航（Affix）插件 问题与解决方式
key: 20150712
tags: Bootstrap affix Link点击错位 滚动页面时尺寸改变
---

使用 Bootstrap 附加导航（Affix）插件 出现各种问题：记录下解决问题的方式，供大家使用参考，言有不当，欢迎拍砖。

### Link点击后的位置偏移

点击侧导航条时，页面上指定的Link会滚动过高，被顶导航条遮住。这个貌似不是Affix的问题，而是顶导航条固定位置的原因。也可以定义到外层div中，这个自己发挥。

#### Css代码 

```
/* Janky fix for preventing navbar from overlapping */  
h1[id] {  
  padding-top: 80px;  
  margin-top: -45px;  
}
```

 **原理:**

比如想要35px的间隙，不能直接写35px。需要 top padding设置成80px，防止顶导航条遮挡。

然后再设置 top margin 为 -45px，以达到35px的效果。



### 滚动页面时尺寸会改变

还有个问题就是滚动页面，当侧边导航条浮起时会改变尺寸。 

 我的解决方式： 使用 js 来动态更改它的**宽度**

**Html代码      附加导航的代码结构 **

```html
<div class="row">
    <div class="col-xs-3" id="myScrollspy">
        <ul id="affix-ui" class="nav nav-tabs nav-stacked" 
            data-spy="affix" data-offset-top="125">
            <li class="active"><a href="#section-1">第一部分</a></li>
            <li><a href="#section-2">第二部分</a></li>
            <li><a href="#section-3">第三部分</a></li>
            <li><a href="#section-4">第四部分</a></li>
        </ul>
    </div>
    <div class="col-xs-9">内容部分</div>
</div>
```

**JavaScript代码 **

```html
<script>
//初始化
$(function(){
    updateLogin(); //载入数据
});
    
//页面改变事件
window.onresize = function(){
    updateLogin();
}

//更新数据            
function updateLogin() {
    $("#affix-ui").width($(".col-xs-3").width());
};
</script>
```