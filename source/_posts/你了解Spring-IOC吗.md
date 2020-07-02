---
title: 你了解Spring IOC吗
date: 2019-09-19 22:01:25
categories:
- Spring
tags:
- Java
- Spring
- IOC
---
学习过Spring框架的人一定都会听过Spring的IOC（控制反转）、DI（依赖注入）这两个概念，对于初学Spring的人来说，总觉得IOC 、DI这两个概念是模糊不清的，是很难理解的。今天和大家分享一下我对IOC的理解。
<!-- more -->
## IOC是什么
IOC（Inversion of Control），即”控制反转”，不是什么技术，而是一种设计思想，是Spring core最核心的部分。IOC使你从繁琐的对象交互中解脱出来，进而专注与对象本身，更近一步突出面向对象。要了解IOC需要先了解软件设计的一个重要思想--依赖注入（Dependency Injection）。
## DI举例
假如我们要设计一个行李箱，如图2-1：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g757454au1j20kq03aweh.jpg"></div>
<center>图 2-1 行李箱依赖关系</center>

这些对象之间存在一条链式的依赖关系，假如轮子的尺寸一改，那么依赖它的底盘也得改，底盘一改那么箱体也得改，同理行李箱也得改。这么一看所有的设计都得改，真是要了老命了。
为了更加形象地表面，下面用代码来进行说明如图2-2：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g758sq1bcxj217w0kiq3r.jpg"></div>
<center>图 2-2 行李箱依赖关系伪代码</center>

如果把Tire的size改成动态可变的，由于依赖关系导致依赖链上的每个对象的构造函数都得加上size参数，这样的设计太难维护了。如图2-3：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g7594lxkzyj217w0knmy2.jpg"></div>
<center>图 2-3 行李箱依赖关系改动伪代码</center>

那么现在我们换一种思路，采用依赖注入的方式实现，如图2-4：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g759dknvn7j20lw03rjre.jpg"></div>
<center>图 2-4 行李箱采用依赖注入方式</center>

这时候我们发现依赖关系倒置了过来，把底层类作为参数传递给上层类，实现上层对下层的“控制”，这就是依赖注入的含义。下面用代码来进行说明如图2-4：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g79axqjuzsj21gu0mhjsy.jpg"></div>
<center>图 2-5 行李箱采用依赖注入伪代码</center>


这时候假如轮子的尺寸一改，我们只需要修改轮子的尺寸即可，不用改变其它对象，这样的代码也便于维护。这种依赖注入的方式会导致你需要你手动new多个对象，但是在Spring中已经帮你管理这些对象的生命周期了，这就是Spring IOC容器。
## IOC、DI、DL的关系
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g79c0nzxccj20r10id3yn.jpg"></div>
<center>图 3-1 IOC、DI、DL的关系图</center>

## 总结
控制反转（Inversion of Control）就是依赖倒置原则的一种代码设计的思路，具体采用的方法就是依赖注入（Dependency Injection），如图4-1：
<div align=center><img src="http://ww1.sinaimg.cn/large/b1bbb565ly1g79c6ow6opj20go07iq5o.jpg"></div>
<center>图 4-1 依赖倒置原则的实现</center>






