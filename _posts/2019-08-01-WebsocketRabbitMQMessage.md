---
layout: post
title: Websocket+RabbitMQ实现消息发送、接收和消息的延时接收
date: 2019-08-01
Author: Applefruits
tags: [Java,Websocket,RabbitMQ,js,Message]
comments: true
---
>本文主要介绍springboot框架下，使用微服务架构的项目，添加消息发送功能过程中所发现和解决的问题，以及此功能的实现过程。    
---
主要逻辑思路：   
>1. 用户登陆系统，建立websocket长链接通道，接收（消费）存储在MQ中的消息。   
2. 给指定的用户群发送消息,后台接收消息，判断消息接收方是否在线。  
 * 在线：直接通过websocket发送到前台；
 * 离线：存储到RabbitMQ队列中

首先第一步是RabbitMQ的安装，我的MQ是安装在lunix服务器上的，安装教程网上一大堆，这里就不过多介绍了。   
第二是在js中写WebSocket请求代码，与java后台建立长连接通道，代码如下：    
websocket.js:   
```
var ws = function () {
    this.success = false;
    //获取登陆用户
    this.user = util.getCash("loginUser");
    //获取Token
    this.token = util.getCash("token");
    this.socket = null;
    var that = this;
    if(!this.user){
        console.error("获取登录用户信息失败，无法初始化聊天功能");
        return;
    }
    if (typeof (WebSocket) == "undefined") {
        console.error("浏览器不支持websocket,无法启用web即时通讯功能");
        return;
    }
    if(!this.token){
        console.error("获取令牌信息失败，无法初始化聊天功能");
        return;
    }
    //格式化token(去掉Bear)
    this.token = this.token.split(" ")[1];
    this.socket = new WebSocket(url.sc_message.chat+"?token=" + this.token);
    this.socket.onopen = function () {
        that.success = true;
        console.log("即时通讯已打开");
    };
    this.socket.onmessage = function (msg) {
        var data = JSON.parse(msg.data);
        if(typeof(data.type)!='undefined'){
            //发送完消息
            return;
        }
        console.log("即时通讯收到消息："+data);
        //弹出消息后，点击确定，重新发送消息给后台，更改此消息在数据库的状态为已读
        alert(data);//alert保证同步，只有在点击确定后才能发送已读状态给后台
        var body = {
            data: {
                sendid: that.getId(),
                sendName: data.recName,
                recid: parseInt(data.sendid),
                status: 1,//已读
            }
        };
        that.send(JSON.stringify(body));
    };
    //关闭事件
    this.socket.onclose = function () {
        console.log("即时通讯已关闭");
    };
    //发生了错误事件
    this.socket.onerror = function (e) {
        console.log("即时通讯发生了错误，聊天功能已不可用" + JSON.stringify(e));
    };
}
//向后台发送消息
ws.prototype.send = function (msg) {
    this.socket.send(msg);
};
```   
在需要消息发送功能的业务js文件中，使用以下代码调用上面的js文件。    
```
var body = {
	data: {
		messageTitle: '',
		messageType: 6,
		receviceId: 1，2，3,//三个用户ID
		readFlag: 1,
		messageDetail: '你好，请接收消息'
	}
};
var webchat = null;
webchat = new ws(body);
```   
接下来介绍Java后台的代码逻辑：    
```
package cn.gov.scmessage.websocket;

import cn.gov.admin.model.AllUser;
import cn.gov.message.model.RabbitInfo;
import cn.gov.message.model.SysMessage;
import cn.gov.message.model.WebSocketMessageBody;
import cn.gov.message.service.SysMessageService;
import cn.gov.message.util.DictUtil;
import cn.gov.message.util.PermissionUtil;
import cn.gov.message.util.SpringUtil;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.rabbitmq.client.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.List;

@ServerEndpoint(value = "/chat")//WebSocket.js文件请求的接口
@Component
public class WebSocketServer {
    private static final Logger logger = LoggerFactory.getLogger(WebSocketServer.class);
    public static ConcurrentHashMap<Integer, WebSocketServer> webSocketSet = new ConcurrentHashMap<>();
    private Session session;
    //队列名称前缀
    private final static String QUEUE_NAME = "MJ_QUEUE";
    private Integer id;
    //用户的姓名，不是登录用的那个名称
    private String name;
    //保存消息用
    public static SysMessageService sysMessageService;

    /**
     * @return void
     * @Description //TODO webSocket.js初期连接（建立长连接），消费还没有消费的消息
     * @Param [session]
     */
    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        Map<String, List<String>> params = session.getRequestParameterMap();
        try {
            if (params == null || !params.containsKey("token")) {
                //认证失败，不能建立连接
                session.close();
                return;
            }
            /*用户连接时根据token解析出用户id和其他详细信息*/
            PermissionUtil permissionUtil = (PermissionUtil) SpringUtil.getBean("permissionUtil");
            AllUser allUser = permissionUtil.getLoginUser(params.get("token").get(0));
            if (allUser == null) {
                //该token认证失败，无法连接
                if (session != null) {
                    try {
                        logger.debug("用户未登录，不能连接到websocket服务器！");
                        session.close();
                    } catch (Exception e) {
                        //关闭出错可以忽略
                    }
                }
                return;
            }
            //获取当前登录用户的用户姓名（不是登录的用户名）
            this.name = allUser.getSysUser().getRealName();
            this.id = allUser.getSysUser().getId();
            //如果websocket中有当前登陆用户，则移除
            if (webSocketSet.containsKey(this.id)) {
                webSocketSet.remove(this.id);
            }
            //添加当前用户到websocket队列
            webSocketSet.put(this.id, this);
            logger.debug("有新连接加入" + this.id);
            //查询当前登陆用户是否有消息未接收
            String queue = QUEUE_NAME + "_" + this.id;
            //如果有消息未接收，则用户初期登陆即消费rabbitMQ中此用户的所有消息--消息离线发送存储，用户上线接收
            reviceInfoToMQ(queue);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * @return void
     * @Description //TODO webSocket连接关闭，清除当前登陆用户
     */
    @OnClose
    public void onClose() {
        logger.debug("连接关闭");
        if (this.id != null) {
            webSocketSet.remove(this.id);
        }
    }

    /**
     * @return void
     * @Description //TODO 接收前台发送消息的响应事件
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        logger.debug("来自客户端消息:" + message);
        //收到消息之后先记录消息到数据库
        WebSocketMessageBody<SysMessage> body = new WebSocketMessageBody();
        ObjectMapper mapper = new ObjectMapper();
        List<String[]> userList = new ArrayList();
        String[] rereceiverArr = null;
        try {
            body = (WebSocketMessageBody<SysMessage>) mapper.readValue(message, body.getClass());
            SysMessage sysMessage = mapper.readValue(mapper.writeValueAsString(body.getData()), SysMessage.class);
            //消息接收人有多个
            if (sysMessage.getReceviceId() != null) {
                rereceiverArr = sysMessage.getReceviceId().split(",");
            }
            //设置消息信息
            PermissionUtil permissionUtil = (PermissionUtil) SpringUtil.getBean("permissionUtil");
            AllUser allUser = permissionUtil.getLoginUser(session.getRequestParameterMap().get("token").get(0));
            Integer sendId = allUser.getSysUser().getId();//获取发送者ID（当前用户）
            sysMessage.setSendId(sendId);
            sysMessage.setCreator(sendId);
            //readflag:1未读；2已读
            if (sysMessage.getReadFlag() != 1) {//消息确认--接收消息，用户确认已读，修改状态为已读
                //收到信息后更新状态为已读
                sysMessage.setUpdater(sendId);
                sysMessage.setUpdateTime(new Date());
                sysMessageService.undateMsgInfo(sysMessage);
            } else {
                //设定发送者的姓名
                sysMessage.setSendName(this.getName());
                sysMessage.setCreateTime(new Date());
                //聊天信息保存到数据库中；保存成功获取此条数据的id
                Integer zId = 0;
                Integer col = sysMessageService.saveMsgInfo(sysMessage);
                if (col != 1) {//保存失败
                    return;
                } else {//保存成功，获取数据库中此条消息ID--用作区分
                    zId = sysMessage.getId();
                }
                //给消息接收者发送消息
                userList.add(rereceiverArr);
                sendAppointInfo(userList, zId + ":" + sysMessage.getMessageDetail(), 1);
                return;
            }
        } catch (Exception e) {
            e.printStackTrace();
            logger.debug("解析聊天信息出错", e);
        }
    }

    /**
     * @return void
     * @Description //TODO 发送消息到MQ
     * @Param [qunue_name]
     * @Param [messageDetail]
     */
    public void sendInfoToMQ(String qunue_name, String messageDetail) {
        try {
            ConnectionFactory con = new ConnectionFactory();
            //设置连接信息
            con.setHost(SpringUtil.getBean(RabbitInfo.class).getHost());//${spring.rabbitmq.host}192.168.1.123
            con.setPort(SpringUtil.getBean(RabbitInfo.class).getPort());//"${spring.rabbitmq.port}5672
            con.setUsername(SpringUtil.getBean(RabbitInfo.class).getUsername());//${spring.rabbitmq.username}admin
            con.setPassword(SpringUtil.getBean(RabbitInfo.class).getPassword());//${spring.rabbitmq.password}admin
            //创建链接
            Connection connection = con.newConnection();
            //创建通道
            Channel channel = connection.createChannel();
            System.out.println(qunue_name + "---消费者队列通道已连接");
            //创建队列
            channel.queueDeclare(qunue_name, false, false, false, null);
            System.out.println("Producer Send '" + messageDetail + "'");
            //存储消息
            channel.basicPublish("", qunue_name, null, messageDetail.getBytes("UTF-8"));
            //存储完成后关闭通道和链接
            channel.close();
            connection.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * @return void
     * @Description //TODO 从MQ接收消息
     * @Param [queue]
     */
    public void reviceInfoToMQ(String queue) {
        try {
            //1.创建一个ConnectionFactory连接工厂connectionFactory
            ConnectionFactory connectionFactory = new ConnectionFactory();
            //2.通过connectionFactory设置RabbitMQ所在IP等信息
            connectionFactory.setHost(SpringUtil.getBean(RabbitInfo.class).getHost());
            connectionFactory.setPort(SpringUtil.getBean(RabbitInfo.class).getPort());
            connectionFactory.setUsername(SpringUtil.getBean(RabbitInfo.class).getUsername());
            connectionFactory.setPassword(SpringUtil.getBean(RabbitInfo.class).getPassword());
            //创建一个新的连接
            Connection connection = connectionFactory.newConnection();
            //创建一个通道
            Channel channel = connection.createChannel();
            //声明要关注的队列
            channel.queueDeclare(queue, false, false, false, null);
            System.out.println("Customer Waiting Received messages");

            //DefaultConsumer类实现了Consumer接口，通过传入一个频道，
            // 告诉服务器我们需要那个频道的消息，如果频道中有消息，就会执行回调函数handleDelivery
            Consumer consumer = new DefaultConsumer(channel) {
                public void handleDelivery(String consumerTag, Envelope envelope,
                                           AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    List<String[]> userList = new ArrayList();
                    String message = new String(body, "UTF-8");

                    PermissionUtil permissionUtil = (PermissionUtil) SpringUtil.getBean("permissionUtil");
                    AllUser allUser = permissionUtil.getLoginUser(session.getRequestParameterMap().get("token").get(0));
                    Integer sendId = allUser.getSysUser().getId();//获取发送者ID（当前用户）
                    String[] receiverId = {String.valueOf(sendId)};
                    userList.add(receiverId);//接收在mq存储的信息的用户
                    sendAppointInfo(userList, message, 2);
                    System.out.println("Customer Received '" + message + "'");
                }
            };
            //自动回复队列应答 -- RabbitMQ中的消息确认机制
            channel.basicConsume(queue, true, consumer);
            TimeUnit.SECONDS.sleep(1);
            channel.close();
            connection.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @OnError
    public void onError(Session session, Throwable error) {
        logger.debug("连接出错");
        Iterator it = webSocketSet.entrySet().iterator();
        //移除出错的websocket对象
        while (it.hasNext()) {
            Map.Entry entry = (Map.Entry) it.next();
            if (session == entry.getValue()) {
                webSocketSet.remove(entry.getKey());
            }
        }
    }

    /**
     * 给当前用户发送消息
     *
     * @param message 消息内容
     * @throws IOException 发送消息出错时抛出异常
     */
    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }

    /**
     * 群发消息，给所有在线的用户发送消息
     *
     * @param message 需要发送的消息
     */
    public static void sendInfo(String message) {
        logger.debug("群发消息：" + message);
        Iterator it = webSocketSet.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry entry = (Map.Entry) it.next();
            try {
                webSocketSet.get(entry.getKey()).sendMessage(message);
            } catch (IOException e) {
                //群发消息时有发送不成功的，忽略，尽可能发送给全部用户
                logger.error("sendInfo1群发消息出错--------------------", e);
            }
        }
    }

    /**
     * 给指定的列表用户群发消息
     *
     * @param userList 用户id列表
     * @param message  需要发送的消息
     */
    public void sendAppointInfo(List<String[]> userList, String message, Integer type) {
        //给指定的列表用户群发消息
        System.out.println("给指定的列表用户群发消息");
        String[] idArr = userList.get(0);
        for (String ids : idArr) {
            int id = Integer.parseInt(ids);
            if (webSocketSet.containsKey(id)) {
                try {
                    webSocketSet.get(id).sendMessage(message);
                    if (type == 1) {
                        String queue = "APP_" + QUEUE_NAME + "_" + id;
                        sendInfoToMQ(queue, message);
                    }
                } catch (IOException e) {
                }
            } else {
                //用户不在线时发送给mq
                String queue = QUEUE_NAME + "_" + id;
                sendInfoToMQ(queue, message);
                String queueApp = "APP_" + QUEUE_NAME + "_" + id;
                sendInfoToMQ(queueApp, message);
            }
        }
    }

    /**
     * 关闭连接，用户退出websocket
     */
    public void close() {
        if (this.session != null) {
            try {
                this.session.close();
            } catch (Exception e) {

            } finally {
                this.session = null;
            }
        }
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public boolean isAppUser() {
        return isAppUser;
    }

    public void setAppUser(boolean appUser) {
        isAppUser = appUser;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```   
存储RabbitMQ连接信息的类：   
```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import javax.persistence.*;
import java.util.Date;
@Component
public class RabbitInfo {
    @Value("${spring.rabbitmq.host}")
    private String host;
    @Value("${spring.rabbitmq.port}")
    private Integer port;
    @Value("${spring.rabbitmq.username}")
    private String username;
    @Value("${spring.rabbitmq.password}")
    private String password;
}
```
