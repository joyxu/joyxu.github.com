--
title: 在安卓中Webkit如何和javascript通信
date: 2017-03-06 20:38:20
tags: Android
---

完整的例子可以从[这里](joyxu.github.com/attachment/JSHome.zip)下载。

##Webkit中调用javascript##
 
假设javascript中存在如下函数：
    
    function resetApplication = function(param){
        ......
    }

在webview中，可以直接通过loadUrl函数调用改方法

    mWebView.loadUrl("javascript:resetApplication('" + buildAppJson() + "')");	
##javascript中调用Webkit##

1   首先要将一个通信的对象暴露给javascript,该对象可以通过addJavascriptInterface函数暴露给javascript

        mJSInterface = new JavaScriptInterface(this , mJSBridgeHandler);
        mWebView.addJavascriptInterface( mJSInterface , "JSBridge");  

2   在之前的JavaScriptInterface对象中添加一个public的方法，让javascript调用

        public void startApp(final String appname)//参数按需修改

3   之后，就可以在javascript的代码中，通过如下语句调用startApp函数了。

        JSBridge.startApp(....);
