---
title: "JavaScript 动态处理由js生成的HTML样式"
date: 2018-11-17T16:04:18+08:00
tags: 
    - JS
    - DOM
categories: 
    - 前端
    - JS
---

> 场景：有一个界面是由其他js动态加载出来的，这个js无法被修改到，但是他的样式不符合当前使用的界面。同时由于界面是H5在移动端显示，所以只能动态适配，不能写死。

## 尝试

我首先想到的方案就是通过js动态为需要适配的div修改样式（`生成出来的布局中的id和css调用是固定的`）。于是通过**DEBUG**拿到需要的id和css名称，在js中拿到元素修改。但是运行后发现没有效果，一看原来在js执行时，该布局未被加载出来，所以拿不到id进行设置。另一种思路是在布局加载完成后进行样式设置，不过这个局限于加载布局的js存在加载完成的回调方法。

## 解决

在Google了一番后，发现了一个监听方法，这个方法可以监听指定元素的DOM树节点变化。于是在这个监听方法中监听需要修改的样式，当通过id拿到的元素的高度(或者其他可以辨别该元素已经被创建的特征)不为`NULL`时，则说明该元素已经被创建，这个时候再进行CSS的设置就有效果了。代码如下：
```javascript
$('body').on('DOMSubtreeModified', () => {
    var div = $('#select_content').height();
    if (div != null) {
        $('#select_content').css("top", "-8px");
        $('#select_bottomBtn').css("top", "-68px");
        $('.cmp-select-basicDiv').css("top", "107%");
    }
});
```