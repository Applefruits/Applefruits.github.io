---
layout: post
title: js校验密码复杂度
date: 2019-08-08
Author: Applefruits
tags: [Html,js,PasswordCheck]
comments: true
---
### eg.1 密码中必须包含大小写 字母、数字、特称字符，至少8个字符，最多30个字符
```
var pwdRegex = new RegExp('(?=.*[0-9])(?=.*[A-Z])(?=.*[a-z])(?=.*[^a-zA-Z0-9]).{8,30}');
if (!pwdRegex.test('password')) {
　　alert("您的密码复杂度太低（密码中必须包含大小写字母、数字、特殊字符），请重试！");
}
```
### eg.2 密码中必须包含字母（不区分大小写）、数字、特称字符，至少8个字符，最多30个字符
```
var pwdRegex = new RegExp('(?=.*[0-9])(?=.*[a-zA-Z])(?=.*[^a-zA-Z0-9]).{8,30}');
if (!pwdRegex.test('password')) {
　　alert("您的密码复杂度太低（密码中必须包含字母、数字、特殊字符），请重试！");
}
```
### eg.3 密码中必须包含字母（不区分大小写）、数字，至少8个字符，最多30个字符
```
var pwdRegex = new RegExp('(?=.*[0-9])(?=.*[a-zA-Z]).{8,30}');
if (!pwdRegex.test('password')) {
　　alert("您的密码复杂度太低（密码中必须包含字母、数字），请重试！");
}
```
 参考：https://www.cnblogs.com/goding/p/10224084.html
