---
layout: page
title: 在浏览器中输入URL到获取内容经历了什么？
categories:
    - frontend
header:
    image_fullwidth: chess.jpg
    caption: Photograph by Drolexandre
    caption_url: http://commons.wikimedia.org/wiki/File:Echecs_chinois.JPG
---
　　用户在浏览器中输入了一个URL之后，通常会经历如下一些步骤：

1. 在浏览器中输入一个URL；
2. 浏览器开始为输入的域名查找IP地址,DNS过程如下：
	
	* 浏览器缓存：浏览器在浏览网页的时候，通常会对浏览记录进行保存（可配置保存时间），时间不固定，通常在2-30分钟之间；
	* OS缓存 ： 操作系统的缓存中查找；
	* 路由器缓存：请求继续发送到路由器，拥有缓存配置的路由器也可以对记录进行一定的缓存；
	* ISP DNS 缓存 ：通常情况下，运营商提供的DNS 服务器中查找；
	* 递归查找 ： ISP的服务器开始递归查找，也就是从根服务器开始查找，先找.com返回一个IP，再去找baidu，再返回一个IP，直到找到为止；
3. 浏览器发送HTTP请求到web服务器；
4. 服务器发送一个永久重定向的301响应；

	* 当发送http://facebook.com的时候，会发送一个301重定向，浏览器会再次发送http://www.facebook.com的请求，这里面有一个有趣的问题，为何不直接指向同一个服务器，而要使用重定向呢？因为搜索引擎会将http://www.facebook.com和http://facebook.com当成两个网站，从而降低访问量排名；另外，相同内容对应多个URL缓存不友好；
	
	![放个图片](images/shawn_world_map.jpg)   
 	
5. 浏览器发送重定向请求；
6. 服务器“处理”请求



