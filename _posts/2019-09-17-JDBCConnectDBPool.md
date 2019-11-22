---
layout: post
title: Java项目用JDBC连接池连接数据库
date: 2019-11-22
Author: Applefruits
tags: [Java,JDBC,DB,Pool]
comments: true
---
>本文的背景是，客户要求使用webservice编写后台服务端接口，在不使用Spring框架的前提下，使用原生的JDBC连接数据库，故写了如下一个例子，供以后类似项目使用借鉴。

首先，使用mysql jar包 [mysql-connector-java-5.1.42.jar](/_posts/res/mysql-connector-java-5.1.42.jar)，在项目中引入jar。

然后在项目中引入jar包并创建文件：数据库创建连接文件JdbcPool；数据库获取连接文件JDBCUtil。文件目录结构如下：![](/_posts/img/20191122.png)

JDBCUtil.java：
```
/*
 * To change this license header, choose License Headers in Project Properties.
 * To change this template file, choose Tools | Templates
 * and open the template in the editor.
 */
package basic;

import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class JDBCUtil {

    private static JdbcPool pool = new JdbcPool();

    /**
     *  * 获取资源
     */
    public static Connection getConnection() throws SQLException {
        return pool.getConnection();
    }

    /**
     *  * 关闭资源  * @param resultSet 查询返回的结果集，没有为空  * @param statement     *
     * @param connection
     */
    public static void close(ResultSet resultSet, Statement statement,
            Connection connection) {
        if (resultSet != null) {
            try {
                resultSet.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
            resultSet = null;
        }

        if (statement != null) {
            try {
                statement.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }

        if (connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
    }

}

```
JdbcPool.java:
```
/*
 * To change this license header, choose License Headers in Project Properties.
 * To change this template file, choose Tools | Templates
 * and open the template in the editor.
 */
package basic;

import java.io.InputStream;
import java.io.PrintWriter;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.LinkedList;
import java.util.Properties;
import java.util.logging.Logger;
import javax.sql.DataSource;

/**
 *  * 原生的jdbc连接池
 */
public class JdbcPool implements DataSource {

    /**
     *  * @Field: listConnections 使用LinkedList集合来存放数据库链接，
     * 由于要频繁读写List集合，所以这里使用LinkedList存储数据库连接比较合适  
     */
    private static LinkedList<Connection> listConnections = new LinkedList<Connection>();
    private static Connection conn;

    static {
        try {
//            String driver = "com.mysql.jdbc.Driver";
//            String url = "jdbc:mysql://172.19.128.38:3306/ydy?characterEncoding=UTF-8&amp;autoReconnect=true&amp;autoReconnectForPools=true";
//            String username = "root";
//            String password = "123456";
            String driver = "com.mysql.jdbc.Driver";
            String url = "jdbc:mysql://10.27.20.253:3306/ydy?characterEncoding=UTF-8&amp;autoReconnect=true&amp;autoReconnectForPools=true";
            String username = "root";
            String password = "sdhm20161028";
            // 数据库连接池的初始化连接数大小
            int jdbcPoolInitSize = 6;
            // 加载数据库驱动
            Class.forName(driver);
            for (int i = 0; i < jdbcPoolInitSize; i++) {
                conn = DriverManager.getConnection(url, username, password);
                // 将获取到的数据库连接加入到listConnections集合中，listConnections集合此时就是一个存放了数据库连接的连接池
                listConnections.add(conn);
            }
        } catch (Exception e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    /*
	 *  * 获取数据库连接
     */
    @Override
    public Connection getConnection() throws SQLException {
        // 如果数据库连接池中的连接对象的个数大于0
        if (listConnections.size() > 0) {
            // 从listConnections集合中获取一个数据库连接
            //System.out.println("listConnections数据库连接池大小是：" + listConnections.size());
            //返回Connection对象的代理对象
            return (Connection) Proxy.newProxyInstance(JdbcPool.class.getClassLoader(), conn.getClass().getInterfaces(),
                    new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    if (!method.getName().equals("close")) {
                        //System.out.println("获取到连接池连接对象......" + conn);
                        return method.invoke(conn, args);
                    } else {
                        // 如果调用的是Connection对象的close方法，就把conn还给数据库连接池
                        listConnections.add(conn);
                        //System.out.println(conn+ "，还给数据库连接池了");
                        //System.out.println("listConnections数据库连接池大小为"+ listConnections.size());
                        return null;
                    }
                }
            });
        } else {
            throw new RuntimeException("对不起，数据库忙");
        }
    }

    //省略@Override方法
    @Override
    public PrintWriter getLogWriter() throws SQLException {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {
        // TODO Auto-generated method stub

    }

    @Override
    public void setLoginTimeout(int seconds) throws SQLException {
        // TODO Auto-generated method stub

    }

    @Override
    public int getLoginTimeout() throws SQLException {
        // TODO Auto-generated method stub
        return 0;
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        // TODO Auto-generated method stub
        return null;
    }
}

```
调用方式：
```
private int ServerIDCheckBloodRecord(int intServerID) {
    //判断是否有数据
    int execute = 0;
    try {
        connection = JDBCUtil.getConnection();
        String sql = "SELECT count(*) as ct FROM blood_record WHERE server_id=? AND del_flag=?";
        pstmtCheck = connection.prepareStatement(sql);
        pstmtCheck.setInt(1, intServerID);
        pstmtCheck.setInt(2, Constant.delFlag_N);
        resultSet = pstmtCheck.executeQuery();
        while(resultSet.next()){
        execute = resultSet.getInt("ct");
        }
        JDBCUtil.close(resultSet, pstmtCheck, connection);
    } catch (SQLException e) {
        System.out.println("异常提醒：" + e);
    }
    return execute;
}
```
