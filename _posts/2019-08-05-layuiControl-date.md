---
layout: post
title: layui时间控件实现选择一段时间内数据
date: 2019-08-05
Author: Applefruits
tags: [Layui,Date,Js]
comments: true
---
>本文介绍Layui前端框架中的时间控件，包括选择一个日期时间段后对控件的制御,选择开始时间后，结束时间只能选择比开始时间晚的时间点

html代码：
```
<div class="layui-form-item informationEssential">
    <div class="layui-inline">
        <label class="layui-form-label">开工时间</label>
        <div class="layui-input-inline datecheck_wrapper">
            <input type="text" name="projectStartTime" id="projectStartTime" lay-verify="date" placeholder="请选择日期"
                   autocomplete="off" class="layui-input">
        </div>
    </div>
    <div class="layui-inline">
        <label class="layui-form-label">竣工时间</label>
        <div class="layui-input-inline datecheck_wrapper">
            <input type="text" name="projectEndTime" id="projectEndTime" lay-verify="date" placeholder="请选择日期"
                   autocomplete="off" class="layui-input">
        </div>
    </div>
</div>
```
js代码：
```
//获取当前时间
$(function () {
  layui.use(['layer', 'table', 'form', 'upload', 'laydate'], function () {
    var form = layui.form;
    var laydate = layui.laydate;
    var now = changeDate();
    var start = laydate.render({
        elem: '#projectStartTime', //开工时间
        min: now,
        done: function (value, date, endDate) {
            end.config.min = {
                year: date.year,
                month: date.month - 1,
                date: date.date
            };
        },
    });
    var end = laydate.render({
        elem: '#projectEndTime', //竣工时间
        btns: ['clear', 'confirm'],  //clear、now、confirm
        done: function (value, date, endDate) {
            start.config.max = {
                year: date.year,
                month: date.month - 1,
                date: date.date
            };
        },
    });
  });
})
```
日期控件赋值：
```
$("#projectStartTime").val(result.data.projectStartTime);
```
js引用：
```
<script src="../../js/jquery.js" type="text/javascript" charset="utf-8"></script>
<script src="../../layui/layui.js" type="text/javascript" charset="utf-8"></script>
<script src="../../js/horizontal-timeline/horizontal-timeline.js" type="text/javascript"></script>
```
