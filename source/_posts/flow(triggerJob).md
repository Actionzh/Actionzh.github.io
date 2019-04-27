---
title: flow(TriggerJob)流程
date: 2019-03-29 10:01:16
tags:
---
1. customerJourneyTriggerJob.doWork(message)
根据routingKey得出eventType和actionType根据名称拼接处QueueWorker的实现bean
2. 调用QueueWorker.work方法
  根据body得出EventDTO，根据dto的event得出EventTriggerCallback接口的实现，
3. 调用EventTriggerCallback.onTrigger
  根据contactId和eventid 生成CustomerJourneyContext，并调用
4. workflowTrigger.trigger
  根据eventdto的event查出所有customerJourney，验证其condition是否满足，满足则发送p2PMessageProducer.send(EXECUTOR_QUEUE, context) mq消息进行处理
  


