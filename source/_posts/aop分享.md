---
title: AOP基础及原理介绍
date: 2019-04-17 00:21:02
categories: spring
tags: [aop]
keywords: aop,框架
---

## 问题
1. 面向切面编程的理解?  
AOP的本质是在一系列纵向的控制流程中，把那些相同的子流程提取成一个横向的面

<!--more--> 

2. 单个切面执行顺序结果是什么？俩个切面重合的时候呢?  
![uml类图](/images/articleImage/actionOrder.jpg)

3. jdk代理和cglib代理的区别?  
jdk面向接口代理,不需要依赖第三方库  
cglib面向类代理,如果没有实现接口会选择cglib代理,效率更快  

4. aop同一个类里调用是否生效?  
特殊情况下可以生效,设置exposeProxy属性为true 暴露代理,调用的时候从上下文获取代理  

### 基础概念
![基础概念思维导图](/images/articleImage/concept.png)

### springAop原理
aop简版uml类图
![uml类图](/images/articleImage/uml.png)
aop创建代理和方法执行流程图
![流程图](/images/articleImage/proxyFloat.png)

### mybaties aop
![mybaties uml](/images/articleImage/mybaties.png)
* MapperRegistry:addMappers方法将包名下每个mapper类创建一个MapperProxyFactory,放入map中
* MapperProxyFactory:创建一个mapperProxy代理类
* MapperProxy:实现了InvocationHandler主要调用MapperMethod的execute
* MapperMethod:创建需要三个参数：DAO接口本身，方法类，Configuration对象 内部新建SqlCommand，MethodSignature俩个类

### 自己实现aop
问题:自己实现一套类似于spring或者mybaties的aop 哪些类是必须的?
* 创建代理类
* 一个methodInvoce的实现类 当作拦截器
* 过滤工具类 用来匹配切点
* 解析xml或者注解的工具类
* [详见代码](https://github.com/zhangyan1/circle)
