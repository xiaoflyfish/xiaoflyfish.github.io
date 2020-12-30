---
title: SpringMVC
categories: 
- Spring
---

# 基本原理

<img src="https://xiaoflyfish.oss-cn-beijing.aliyuncs.com/image/20201230115144.png" alt="img" style="zoom:33%;" />

1.客户端（浏览器）发送请求，直接请求到DispatcherServlet

2.DispatcherServlet根据请求信息调用`HandlerMapping`，解析请求对应的Handler

3.解析到对应的`Handler`后，开始由`HandlerAdapter`适配器处理

4.HandlerAdapter会根据`Handler`来调用真正的处理器开处理请求，并处理相应的业务逻辑

5.处理器处理完业务后，会返回一个`ModelAndView`对象，Model是返回的数据对象，`View`是个逻辑上的View

6.ViewResolver会根据逻辑`View`查找实际的View

7.DispatcherServlet把返回的`Model`传给View

8.通过View返回给请求者（浏览器）