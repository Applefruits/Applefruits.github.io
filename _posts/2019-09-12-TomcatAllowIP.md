---
layout: post
title: 利用Tomcat限制IP登陆系统或调用接口
date: 2019-09-12
Author: Applefruits
tags: [Tomcat,Permission,lunix]
comments: true
---
### 本文针对客户要求为：限制Ip登陆，只允许客户指定的IP调用接口
找到tomcat目录的/conf/server.xml文件，在<host>标签上方添加如下规格的语句
```
<!-- 只允许212.173.36.131访问 -->
<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="212.173.36.131" deny=""/>
```
