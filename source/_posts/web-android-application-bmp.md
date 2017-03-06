---
title: Web显示Android应用程序图片
date: 2017-03-06 20:38:44
tags: Android
---

Android中应用程序的图标都是各个程序的资源或者系统的资源，
已经不是一副图片，如果一个应用是基于web的而且要显示系统
或者应用的图标改怎么做呢？

##首先说说web中显示图片的方法:

1   img的src中直接指向图片的uri

2   img的src直接是一副图片的[base64编码](http://baike.baidu.com/view/469071.htm)

###在这里要说的就是通过把图片编码成base64，之后显示，具体做法：

1   通过Android内部的loadIcon方法得到应用的icon，但是是一个drawable的对象，
参考代码如下
    
        application.icon = info.activityInfo.loadIcon(manager);

2   得到drawable对象之后，通过bitmap和Base64提供的方法，转换成字符串

    
        private String encodeIcon(Drawable icon){
            Drawable ic = icon;
	        if(ic !=null){
		        BitmapDrawable bitDw = ((BitmapDrawable) ic);
		        Bitmap bitmap = bitDw.getBitmap();
		        ByteArrayOutputStream stream = new ByteArrayOutputStream();
		        bitmap.compress(Bitmap.CompressFormat.JPEG, 90, stream);
		        byte[] bitmapByte = stream.toByteArray();
		        return Base64.encodeToString(bitmapByte, Base64.NO_WRAP);
	        }
	        return "";
	    }

3   得到Base64的字符串之后，通过json传给web

        JSONArray jsonArray = new JSONArray();
		for(ApplicationInfo app:mApplications){
			JSONObject jsonObject = new JSONObject();
			jsonObject.put("icon", encodeIcon(app.icon));
			jsonArray.put(jsonObject);
		}
		return jsonArray.toString();


