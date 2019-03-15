2015-07-18-Android adb 启动应用
---
layout:     post
title:      App启动流程分析
subtitle:   App启动流程分析
date:       2015-07-18
author:     YGS
header-img: img/post-bg-def.jpg
catalog: true
tags:
    - Android
    - Adb启动应用
    - Scheme启动
    - 包名/类名启动
---

# Android adb 启动应用
## 通过Scheme启动命令

	在Android中大部分浏览器是不支持Scheme启动应用，这一点没ios好使。
	
	adb -d shell am start -d sinaweibo://xxx -a android.intent.action.VIEW

	带参数命令：
	adb -d shell am start -d sinaweibo://xxx?url="url" -a android.intent.action.VIEW
	
	获取参数方法：getIntent().getData().getQueryParameter("url");

	
## 通过包名/类名启动

	adb shell am start -n com.xxx/com.xxx.xxx
	
	带参数命令：
	adb shell am start -n com.xxx/com.xxx.xxx --es url "url"
	
	获取参数方法：getIntent().getStringExtra("url");

	参数说明：都是 <key> <value> 形式
	--es String类型			--ez boolean类型
	--ei Int类型				--ef float类型
	
	


