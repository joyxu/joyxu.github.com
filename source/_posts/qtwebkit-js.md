---
title: 在QtWebkit中如何和javascript通信
date: 2017-03-06 20:38:06
tags: Qt
---

##QtWebkit中调用javascript##
 
    假设javascript中存在如下函数：
        
        function Response(result,size){
            ......
        }

1   先定义一个用于和javascript通信的类，该类必须继承于QObject,
并添加一个Response的signals，参数和javascript对应，返回值必须为void

        class JSBridge : public QObject {
        Q_OBJECT
        public:
            JSBridge();
            ~JSBridge();

        signals:
            void Response(QString result, int size);
        }

2   在JSBridge的另外一个函数中,就可以调用

        emit Response(QString(result.c_str()), size);

    来调用javascript中的Response函数了。
    
       
##javascript中调用QtWebkit##

1   首先要将一个通信的对象暴露给javascript，可以在QtWebView的某个函数中调用

        m_JSBridge = new JSBridge();
        addToJavaScriptWindowObject(QString("JSBridgeObject"), m_JSBridge);

2   在之前的JSBridge对象中添加一个slot

        void Request(int requestID, QVariantList parameter);//参数按需修改

     修改后JSBridge如下：

        class JSBridge : public QObject {
        Q_OBJECT
        public:
            JSBridge();
            ~JSBridge();

        public slots:
            void Request(int requestID, QVariantList parameter);

        signals:
            void Response(QString result, int size);
        }

3   之后，就可以在javascript的代码中，通过如下语句调用Request函数了。

        JSBridgeObject.Request(request.id, request.param);
