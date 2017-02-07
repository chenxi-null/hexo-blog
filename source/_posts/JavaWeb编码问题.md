title: JavaWeb编码问题
tags:
  - java
categories: []
author: chenxi
date: 2016-12-20 21:49:00
---

## 浏览器端 ##

+ url中的编码。
分为两种情况：url路径和url参数（这里指的是手动在地址栏敲击中文等非ASCII字符）。采用何种字符集由浏览器决定。

+ form表单提交内容的编码。
不论是get还是post请求，采用的都是**“页面指定的编码"**。
指定页面编码有两种方式：
 - html中的meta标签: `<meta charset="utf-8">`
 - 返回本页面时服务端的返回头: `Content-Type:text/html;charset=gbk`


+ ajax请求 
实验证明ajax默认使用的是utf-8编码，和页面默认编码方式无关。
至少对于**JQuery的ajax**请求来说，在IE和Chrome上测试:
 - 通过post请求提交的数据，服务端不做特殊处理也不会产生乱码；
      contentType: "application/x-www-form-urlencoded; charset=utf-8"  两种浏览器都会在加上这一请求头。
 - 而对于get请求的话，两个浏览器的处理方式就不同了:
       chrome中默认使用utf-8编码；而IE的编码方式未知     


+ JS编码：
 - encodeURI两次编码，对应服务端的两次解码：tomcat解码一次加代码中解码一次。
 - +号消失问题：http协议遗留问题，服务端将+号当作空格的编码，解决办法：将加号用%2B替换或者使用encodeURIComponent解码部分内容
 - $.ajax的“data”参数，`data: param`
如果param是js个对象，发送post请求的时候会以encodeURIComponent形式进行编码




## HTTP服务器端（以Tomcat为例）##
浏览器向服务器提交请求内容，无非是使用get或者post两种方式（根据HTTP协议，还存在head，put等请求方式，这里先不讨论），所以服务端的解码方式自然也分两块，一是对于url中参数(queryString)的解码方式，二是对于请求体(postData)中内容的解码方式。

+ Tomcat中对于requestBody中的内容(**PostData**)采用的默认编码为ISO-8859-1，
可在Servlet中设置：`request.setCharacterEncoding("utf-8");·`
或者在提交时通过请求头指定编码类型：`Content-Type: application/x-www-form-urlencoded; charset=UTF-8`
通过这两种方式让Tomcat以我们指定的字符集来解码。

+ 对于URI中携带的参数，Tomcat会采用其默认编码字符集先对**QueryString**进行一次解码， Tomcat7及其之前的版本采用的编码集为ISO-8859-1 ，而到了Tomcat8I默认字符集改成了UTF-8。
可在tomcat的server.xml中配置，修改成自己想要的字符集。
    

### JSP及Servlet API的使用 ###
+ 通常我们在JSP文件的Page指令标签中看到这样的声明：
`<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>`
指定了contentType属性，其效果是客户端如果请求这个jsp，服务器返回的响应报文会添加contentType这一响应头，通过这种方式可是指定当前页面的编码，这样浏览器既会以预期的编码来解析html，也会在提交form表单时使用这一字符集进行编码。

+ 如何在Servlet里正确的设置ContentType

	- 服务端什么都不做
	结果：没有Content-Type这一个http响应头

	- response.setContentType("text/html");  // 没有指定charset
	结果：Content-Type:text/html;charset=ISO-8859-1，默认指定编码为IOS-8859-1，这样包含中文字符的话会出现乱码

	-  response.setCharacterEncoding("utf-8");  // 没有调用HttpServletResponse#setContentType方法
	结果：也没有Content-Type这一个http响应头

	- 要想返回预期的Content-Type，如："Content-Type:text/html;charset=UTF-8", 有两种方式:
	1.
	response.setContentType("text/html"); 
	response.setCharacterEncoding("utf-8");
	2.
	response.setContentType("text/html;utf-8");


