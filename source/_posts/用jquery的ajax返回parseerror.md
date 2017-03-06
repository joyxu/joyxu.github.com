---
title: 用jquery的ajax返回parseerror
date: 2017-03-06 12:20:22
tags:
---

#jquery中使用ajax请求json页面，返回parseerror

原始代码如下：

        $.ajax({
        url: that.indexPage ,
        success: function(data){
            var tips = eval(data);
            ...
        }

结果发现,success的回调函数总是没有被调用,
但是在浏览器中直接访问该页面，是可以得到数据的。
后来，只有加入error的回调函数，看到底出什么错。
添加error的代码如下：

    $.ajax({
        url: that.indexPage ,
        error: function(jqXHR, textStatus, errorThrown){
            alert(textStatus);
            alert( jqXHR.responseText);
        },
        success: function(data){
            var tips = eval(data);
            ...
        }

结果发现textStatus显示的错误是parseerror，jqXHR.responseText
显示的是请求的json的数据，于是google之，发现很多都是再说jquery
会检查返回的数据，如果不是json的话会出错，可是返回的数据是json，
还出错就搞不明白了，但是有一点就是可以修改datatype来看看，于是
试着修改datatype，发现改为html之后就可以啦，ok的代码如下:

    $.ajax({
        url: that.indexPage ,
        dataType: "html",
        error: function(jqXHR, textStatus, errorThrown){
            alert(textStatus);
            alert( jqXHR.responseText);
        },
        success: function(data){
            var tips = eval(data);
            ...
        }
