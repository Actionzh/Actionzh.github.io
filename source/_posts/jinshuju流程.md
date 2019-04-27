---
title: 总结0403(金数据linkflow分析)
date: 2019-04-03 23:40:16
tags:
---
### 1.connect
1.1根据code和state获取token,得到token,refresh_token,和期限等信息->用accesstoken获取profile
1.2将email作为channelId,查数据库是否已存在channeldto
不存在则重新构建channeldto和channedto.property
1.3事务执行channeldto插入数据库，同时将access_token和refresh_token插入redis,accesstoken设置redis过期时间

### 2.getaccesstoken根据channelId
从redis获取accesstoken如果为空，则获取refreshtoken,并从新请求获取token信息，并设置进入redis(refreshToken每次都更新进channel.property的数据库字段中)

### 3.handleSubmit(String channelId, JsonNode responseBody)
1.根据JinshujuSubmitDTO获取FormMappingDTO
2.判断是否是微信过来的用户，曾经关注过的用户取出contact; 未关注过的新建用户创建匿名用户
3.既是匿名用户又没有表单映射，退出
匿名访问情况下先通过表单的值查询是否有已存在用户（通过邮箱和手机号）
4.根据channeldto和submitdto创建EventDTO
5.用channelId和contactDTO调用
==contactFacade.save==：
保存contactDTO，更新渠道账号信息ContactIdentityDTO
循环获取contactdto的event,调用：
==eventService.create(eventDTO)==：
保存event，循环子事件，调用rabbitMq消息

