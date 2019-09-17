---
layout: post
title: Java项目直接用JDBC连接数据库
date: 2019-09-17
Author: Applefruits
tags: [Java,JDBC,DB]
comments: true
---
>本文的背景是，客户要求使用webservice编写后台服务端接口，在不使用Spring框架的前提下，使用原生的JDBC连接数据库，故写了如下一个例子，供以后类似项目使用借鉴。

首先，下载jar包 [mysql-connector-java-5.1.42.jar](/res/mysql-connector-java-5.1.42.jar)

然后在项目中引入jar包并创建文件：数据库连接文件ConnectionManager；数据库配置文件context.xml。文件目录结构如下：[图 ](/img/projectStucture.png)

ConnectionManager：
```
package basic;

import javax.naming.Context;
import javax.naming.InitialContext;
import javax.naming.NamingException;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class ConnectionManager {
    public static Connection conn=null;
    public static ResultSet rs=null;
    public static PreparedStatement pstmt=null;
    public static PreparedStatement pstmt2=null;
    /**
     * @return
     */
    public static Connection getConnection(){
        try {
            Context ct=new InitialContext();
            DataSource ds=(DataSource) ct.lookup("java:comp/env/jdbc/ydy");
            conn=ds.getConnection();
        } catch (SQLException e) {
            e.printStackTrace();
        }catch (NamingException e) {
            e.printStackTrace();
        }
        return conn;
    }
    public static void closeAll(){
        try{
            if(conn!=null){
                conn.close();
            }
            if(pstmt!=null){
                pstmt.close();

            }
            if(rs!=null){
                rs.close();
            }
        }catch(SQLException e){
            e.printStackTrace();
        }
    }
}
```
context.xml:
```
<?xml version='1.0' encoding='utf-8'?>
<!-- The contents of this file will be loaded for each web application -->
<Context>
    <Resource
            name="jdbc/DBName" auth="Container" type="javax.sql.DataSource"
            maxAction="100" maxIdle="30" maxWait="10000"
            username="root" password="123456"
            driverClassName="com.mysql.jdbc.Driver"
            url="jdbc:mysql://127.0.0.1:3306/DBName?characterEncoding=UTF-8"
    />
</Context>
```
调用方式：
```
private int Insert(String strPrgID, int intServerID) {
        //判断添加数据是否成功
        int result = 0;
        PreparedStatement pstmtInsert = null;
        conn = getConnection();
        //SQL
        String sql = "INSERT INTO record (user_id,server_id) VALUES (?,?)";
        try {
            pstmtInsert = conn.prepareStatement(sql);
            pstmtInsert.setString(1, strPrgID);
            pstmtInsert.setInt(2, intServerID);
            result = pstmtInsert.executeUpdate();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }
```
