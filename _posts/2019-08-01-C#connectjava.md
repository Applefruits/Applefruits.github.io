---
layout: post
title: c#连接Java后台，处理返回的数据
date: 2019-08-01
Author: Applefruits
tags: [Java,C#,HttpWebRequest]
comments: true
---
**首先定义共通文件，根据url连接Java后台**
```
class ConntectUtil
{
  public JObject ConsoleApplication(string appID, CustomDataType.HttpMethod method, string strJson)
  {
      String strResponse = "";
      HttpWebRequest request = (HttpWebRequest)HttpWebRequest.Create(new Uri(url + appID));
      //POST或PUT
      switch (method)
      {
          case Method.POST:
              //设置参数
              var jsonParse = JObject.Parse(strJson);
              String jsonParam = "strJson=" + jsonParse.ToString();
              byte[] buff = System.Text.Encoding.UTF8.GetBytes(jsonParam);
              //设置请求信息
              request.Method = CustomDataType.HttpMethod.POST.ToString();
              request.ContentType = "application/x-www-form-urlencoded; charset=UTF-8";
              request.ContentLength = buff.Length;
              Stream stream = request.GetRequestStream();
              stream.Write(buff, 0, buff.Length);
              stream.Dispose();
              WebResponse response = request.GetResponse();
              StreamReader reader = new StreamReader(response.GetResponseStream(), Encoding.UTF8);
              strResponse = reader.ReadToEnd();//利用StreamReader就可以从响应内容从头读到尾
              stream.Close();
              response.Close();
              break;
          case Method.GET:
              WebResponse myResponse = request.GetResponse();
              StreamReader myReader = new StreamReader(myResponse.GetResponseStream(), Encoding.UTF8);
              strResponse = myReader.ReadToEnd();//利用StreamReader就可以从响应内容从头读到尾
              myReader.Close();
              myResponse.Close();
              break;
      }
      return (JObject)JsonConvert.DeserializeObject(strResponse);
  }
}
```
---
**带参数的请求后台获取数据。**  
```
ConntectUtil conntectUtil = new ConntectUtil();

//设置参数
Temp temp = new Temp();
temp.Id = 1;
//格式化成JSON格式
String strJson = JsonConvert.SerializeObject(temp);
//获取返回数据
JObject retJson = commonUtil.ConsoleApplication(ConfigurationManager.AppSettings["url"], CustomDataType.HttpMethod.POST, strJson);
//（第二种写法）JObject retJson = commonUtil.ConsoleApplication(url, POST, strJson);
```
**在请求后台功能中，常常需要对返回的数据进行处理后赋值，本文将讲解两种赋值方法：**   
首先，获取返回的JSON数据  
```
//获取数据
string jsonData = JsonConvert.SerializeObject(retJson.GetValue("data"));
```
1.直接给DataGridView绑定数据
```
//禁止根据返回数据自动生成列
this.DataGridView.AutoGenerateColumns = false;
//绑定数据源
this.DataGridView.DataSource = retJson.GetValue("data");
```
2.转成list后做遍历处理
```
//转化为list
JavaScriptSerializer Serializer = new JavaScriptSerializer();
List<Temp1> objs = Serializer.Deserialize<List<Temp1>>(jsonData);
```
