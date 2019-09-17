---
layout: post
title: Java项目直接用JDBC连接数据库
date: 2019-09-17
Author: Applefruits
tags: [Java,JDBC,DB]
comments: true
---
>本文的背景是，客户要求使用webservice编写后台服务端接口，在不使用Spring框架的前提下，使用原生的JDBC连接数据库，故写了如下一个例子，供以后类似项目使用借鉴。

首先，下载jar包 [mysql-connector-java-5.1.42.jar](/res/mysql-connector-java-5.1.42.jar)，
