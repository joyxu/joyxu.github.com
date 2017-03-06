---
title: Qt QML
date: 2017-03-06 20:38:30
tags: Qt
---

**到写这篇日志的时候，Qt已经是5.0了，Qt用于界面的编写已经从最初的
QtGui -> QtGraphic -> QtQuick/QML**

之前一直在找一个好的GUI开发包，用过底层的Directfb，也用过上层的GTK
,Qt以及当下比较流行的javascript/webkit,简单比较一下优劣吧:

*   Directfb太过于底层，虽然也提供鼠标键盘事件，也有继续Directfb的
脚本，但是始终觉得不是很主流，而且不是所见即所得，觉得人做起来效率
比较低

*   GTK只是编译，移植到平台过，实际的程序还没写过，就不好评论了

*   Qt的话，说起来就比较多了，用QtGui做的话，其实和常见的MFC,VCL做
起来的效率差不多,不是很高

*   javascript/webkit,这个确实很好，效率也高，但是在大多数平台上面，
性能还有待提高,改性能在以下平台跑过：
    
    *   mips 400MHZ,60MB memory for linux(跑完内核+webkit，大概花费50MB，还剩下10MB), linux

    *   A9 1G , 1G memory , android

    *   arm 9 500MHZ , 512MB memory  , WinCE

到现在开始尝试QML之后，发现不但性能有很大提高，特别是GUI的平滑度,而且QML
是基于javascript的扩展，本省就支持javascript，而且开发起来也是所见及所得,
所以现在就特别推荐使用QML来做界面了。

关于QML[请参考这里](http://en.wikipedia.org/wiki/QML)和Qt[请参考这里](http://en.wikipedia.org/wiki/Qt_(framework\) ),可以发现QML也是Qt5.0的大头了。

以后会写出更多的QML的文章。
