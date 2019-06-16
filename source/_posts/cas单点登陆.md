---
title: cas单点登陆
date: 2019-06-16 12:52:05
tags:
---
## **1、结构体系**
在上面的文章中，我们介绍过，在CAS的结构中主要分两部分，一部分是CAS Server，另一部分是CAS Client。

CAS Server：CAS Server 负责完成对用户的认证工作 , 需要独立部署 , CAS Server 会处理用户名 / 密码等凭证(Credentials)。

CAS Client：负责处理对客户端受保护资源的访问请求，需要对请求方进行身份认证时，重定向到 CAS Server 进行认证。（原则上，客户端应用不再接受任何的用户名密码等 Credentials ）。

![1](cas%E5%8D%95%E7%82%B9%E7%99%BB%E9%99%86/1.png)
```
SSO单点登录访问流程主要有以下步骤：
    1. 访问服务：SSO客户端发送请求访问应用系统提供的服务资源。
    2. 定向认证：SSO客户端会重定向用户请求到SSO服务器。
    3. 用户认证：用户身份认证。
    4. 发放票据：SSO服务器会产生一个随机的Service Ticket。
    5. 验证票据：SSO服务器验证票据Service Ticket的合法性，验证通过后，允许客户端访问服务。
    6. 传输用户信息：SSO服务器验证票据通过后，传输用户认证结果信息给客户端。
```

## **2、CAS协议**
CAS协议是一个简单而强大的基于票据的协议，它涉及一个或多个客户端和一台服务器。即在CAS中，通过TGT(Ticket Granting Ticket)来获取 ST(Service Ticket)，通过ST来访问具体服务。

其中主要的关键概念：

TGT（Ticket Granting Ticket）是存储在TGCcookie中的代表用户的SSO会话。
该ST（Service Ticket），作为参数在GET方法的URL中，代表由CAS服务器授予访问CASified应用程序（包含CAS客户端的应用程序）具体用户的权限。

## **3、CAS术语概念**
**1、CAS 系统中的票据： TGC、TGT 、 ST 、 PGT 、 PGTIOU 、 PT 。**
(1)、TGC（ticket-granting cookie）
授权的票据证明，由 CAS Server 通过 SSL 方式发送给终端用户，存放用户身份认证凭证的Cookie，在浏览器和CAS Server间通讯时使用，并且只能基于安全通道传输（Https），是CAS Server用来明确用户身份的凭证。

(2)、TGT（Ticket Grangting Ticket）
TGT是CAS为用户签发的登录票据，拥有了TGT，用户就可以证明自己在CAS成功登录过。TGT封装了Cookie值以及此Cookie值对应的用户信息。用户在CAS认证成功后，CAS生成Cookie（叫TGC），写入浏览器，同时生成一个TGT对象，放入自己的缓存，TGT对象的ID就是Cookie的值。当HTTP再次请求到来时，如果传过来的有CAS生成的Cookie，则CAS以此Cookie值为key查询缓存中有无TGT ，如果有的话，则说明用户之前登录过，如果没有，则用户需要重新登录。

(3)、ST（Service Ticket）
ST是CAS为用户签发的访问某一service的票据。用户访问service时，service发现用户没有ST，则要求用户去CAS获取ST。用户向CAS发出获取ST的请求，如果用户的请求中包含Cookie，则CAS会以此Cookie值为key查询缓存中有无TGT，如果存在TGT，则用此TGT签发一个ST，返回给用户。用户凭借ST去访问service，service拿ST去CAS验证，验证通过后，允许用户访问资源。

(4)、PGT（Proxy Granting Ticket）
Proxy Service的代理凭据。用户通过CAS成功登录某一Proxy Service后，CAS生成一个PGT对象，缓存在CAS本地，同时将PGT的值（一个UUID字符串）回传给Proxy Service，并保存在Proxy Service里。Proxy Service拿到PGT后，就可以为Target Service（back-end service）做代理，为其申请PT。

(5)、PGTIOU（Proxy Granting Ticket I Owe You）
PGTIOU是CAS协议中定义的一种附加票据，它增强了传输、获取PGT的安全性。
PGT的传输与获取的过程：Proxy Service调用CAS的serviceValidate接口验证ST成功后，CAS首先会访问pgtUrl指向的Https URL，将生成的 PGT及PGTIOU传输给proxy service，proxy service会以PGTIOU为key，PGT为value，将其存储在Map中；然后CAS会生成验证ST成功的XML消息，返回给Proxy Service，XML消息中含有PGTIOU，proxy service收到XML消息后，会从中解析出PGTIOU的值，然后以其为key，在Map中找出PGT的值，赋值给代表用户信息的Assertion对象的pgtId，同时在Map中将其删除。

(6)、PT（Proxy Ticket）
PT是用户访问Target Service（back-end service）的票据。如果用户访问的是一个Web应用，则Web应用会要求浏览器提供ST，浏览器就会用Cookie去CAS获取一个ST，然后就可以访问这个Web应用了。如果用户访问的不是一个Web应用，而是一个C/S结构的应用，因为C/S结构的应用得不到Cookie，所以用户不能自己去CAS获取ST，而是通过访问proxy service的接口，凭借proxy service的PGT去获取一个PT，然后才能访问到此应用。

**2、TGT、ST、PGT、PT之间关系**
ST是TGT签发的。用户在CAS上认证成功后，CAS生成TGT，用TGT签发一个ST，ST的ticketGrantingTicket属性值是TGT对象，然后把ST的值redirect到客户应用。
PGT是ST签发的。用户凭借ST去访问Proxy service，Proxy service去CAS验证ST（同时传递PgtUrl参数给CAS），如果ST验证成功，则CAS用ST签发一个PGT，PGT对象里的ticketGrantingTicket是签发ST的TGT对象。
PT是PGT签发的。Proxy service代理back-end service去CAS获取PT的时候，CAS根据传来的pgt参数，获取到PGT对象，然后调用其grantServiceTicket方法，生成一个PT对象。

**其他单点登陆：**
[https://blog.csdn.net/u012394095/article/details/79732224](https://blog.csdn.net/u012394095/article/details/79732224)
