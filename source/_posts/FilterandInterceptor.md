---
title: 拦截器和过滤器
date: 2019-03-20
tags:
---
GenericFilterBean
https://blog.csdn.net/liangxw1/article/details/51095484
RequestInterceptor
https://www.jianshu.com/p/919d066a07aa


Servlet中提供了8个监听器

==一类==:监听三个域对象的创建和销毁的监听器
对象类型	对应的监听器
ServletContext	ServletContextListener
HttpSession	HttpSessionListener
HttpServletRequest	ServletRequestListener

==二类==:监听三个域对象的属性变更的监听器.(属性添加,属性移除,属性替换)
对象类型	对应的监听器
ServletContext	ServletContextAttributeListener
HttpServletRequest	ServletRequestAttributeListener
HttpSession	HttpSessionAttributeListener

==三类==:监听HttpSession对象中的JavaBean的状态的改变.(绑定,解除绑定,钝化和活化)2个
对象类型	对应的监听器
HttpSession	HttpSessionBindingListener(绑定,解除绑定)
HttpSession	HttpSessionActivationListener(钝化和活化)
​

