---
layout: post
title: WebService+Axis项目部署到lunix服务器的Tomcat上
date: 2019-09-17
Author: Applefruits
tags: [Tomcat,WebService,lunix，项目部署]
comments: true
---
>本文描述webservice项目从创建到发布的一系列实现方式

## 一、创建WebService项目
在idea编辑器中创建WebService项目：
1. File-New-Project：![](/_posts/img/20190917-1.png)
2. 选择Java-WebApplication(WebService)-Version(Apache Axis)-Next ![](/_posts/img/20190917-2.png)
3. 填写项目名称和项目保存目录 ![](/_posts/img/20190917-3.png)
4. 项目创建完成，目录结构如下 ![](/_posts/img/20190919-1.png)
5. 添加Tomcat：点击Add Configuration-点击加号-Tomcat Server -local,选择Application server（本地Tomcat位置）-设置Name.

## 二、编辑WebService接口端
1. 在src目录下编辑HelloWorld文件，注意创建Public方法（只有public方法才能作为接口发布）
2. 编辑完成后右键HelloWorld文件，WebServices-Generate Wsdl From Java-选择要发布的接口方法-ok,即可在同级目录下生成.wsdl文件。（如果生成失败可编译一下项目重试）
3. 每新建一个class需要在server-config.wsdd文件中添加一段代码（style="wrapped"代表可以传多个参数）

```
<service name="HelloWorld" provider="java:RPC" style="wrapped" use="literal">
       <parameter name="className" value="example.HelloWorld"/>
       <parameter name="allowedMethods" value="*"/>
       <parameter name="scope" value="Application"/>
       <namespace>http://example</namespace>
   </service>
```

4. 启动Tomcat，在浏览器中回出现index.jsp页面，在地址框输入地址：http://localhost:8080//services，会出现调用链接，复制到soapui测试即可

## 三、WebService项目发布
首先配置发布设置信息，File-projectStucture,具体设置如下图：![](/_posts/img/20190919-2.png)

点击Ok后，回到主页面![](/_posts/img/20190919-3.png),![](/_posts/img/20190919-4.png)

发布完成后会在项目根目录下生成out文件夹，D:\text1\out\artifacts\text1_war_exploded文件夹下的WEB-INF文件夹即为发布文件，至此，发布成功。

## 四、WebService项目部署
本项目部署到lunix服务器上。
1. lunix服务器安装Tomcat，可以找网上教程
2. 在tomcat的webapps目录下新建test文件夹作为本项目的部署路径
3. 复制发布成功的WEB-INF文件夹到test目录下
4. 启动tomcat，放开相应端口权限，在浏览器输入http://114.215.24.88:8088/WS/servlet/AxisServlet 即可查看项目部署内容。

------
注意：
1. 修改conf下的context.xml文件，设置链接数据库信息（注意使用内网IP）
2. conf下的server.xml文件可以做限制IP等操作
