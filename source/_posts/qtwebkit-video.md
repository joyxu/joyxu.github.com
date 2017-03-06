---
title: QtWebkit网页显示视频
date: 2017-03-06 20:37:50
tags: Qt
---

一般来说要在网页中显示视频的话，有一下两种方法：

*   利用现有的tag,比如html5中的video
*   利用现有的插件，比如flash

那么今天要说的是另外一种方法:在网页上面抠洞，
这样处于下面的video就可以显示出来

环境： 

QtWebkit 自定义浏览器

具体的做法： 

重写WebView的paintEvent方法，清空一块区域，
下面是一个例子：


    void CMyWebView::paintEvent( QPaintEvent* event )
    {
        QWebView::paintEvent(event);

        if ( IsVideoWindowVisible() )
        {
            video_win_t videoWindowSize = GetVideoWindowSize();
            
            QPainter p(this);
            p.setCompositionMode( QPainter::CompositionMode_Clear );
            p.fillRect(videoWindowSize.x, videoWindowSize.y, videoWindowSize.w, videoWindowSize.h, Qt::SolidPattern);
        }
    } 
