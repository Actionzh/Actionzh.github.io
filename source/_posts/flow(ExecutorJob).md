---
title: flow(ExecutorJob)流程
date: 2019-03-29 10:01:16
tags:
---
1. CustomerJourneyExecutorJob.doWork
根据body转化出customerJourneyContext，然后调用
2. workflowTrigger.trigger
调用doTrigger,得到(context.getTriggerId(), processDefId, processVariables)调用
3. workflowManager.run(内部为acticiti的runtimeService启动流程)返回processInstanceId,生成CustomerJourneyInstance并入库


#### action的逻辑实现调用部分
看bpmn的xml发现，serviceTask 节点内有activiti:delegateExpression的信息，值为#{TriggerDelegateDispatcher}，#{ActionDelegateDispatcher}联想到spring代码中的delegate，查看代码发现抽象BaseDispatcher的实现有ActionDelegateDispatcher和TriggerDelegateDispatcher证实了想法。

ActionDelegateDispatcher的executeDelegate方法根据customerJourneyProperty.getEvent()得到beanname，从而得到AbstractDelegate的具体实现的bean对象，最终执行delegate.executeDelegate方法，实现自实现的逻辑（每个action的实现逻辑即在此进行调用和实现）


```
<?xml version='1.0' encoding='UTF-8'?> <definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:activiti="http://activiti.org/bpmn" xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI" xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC" xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI" typeLanguage="http://www.w3.org/2001/XMLSchema" expressionLanguage="http://www.w3.org/1999/XPath" targetNamespace="http://www.activiti.org/processdef"> <process id="P9fbf8335-9f0a-4c8e-a1bf-a6c4ec856c1a" isExecutable="true"> <startEvent id="START"/> <sequenceFlow sourceRef="START" targetRef="N-8f2e9577-46e5-4833-8cb6-e0622d531496"/> <serviceTask id="N-8f2e9577-46e5-4833-8cb6-e0622d531496" name="微信" activiti:delegateExpression="#{TriggerDelegateDispatcher}"> <extensionElements> <activiti:field name="parameterMap"> <activiti:string><![CDATA[{"triggerHistoryEvent":false,"channelId":"wx1f7fbd28ac009cd0","conditions":"ANY"}]]></activiti:string> </activiti:field> </extensionElements> </serviceTask> <endEvent id="END-N-d3219b76-e9c7-4a5b-b59e-5342c92f8548"/> <sequenceFlow id="END-N-d3219b76-e9c7-4a5b-b59e-5342c92f8548-flow" sourceRef="N-d3219b76-e9c7-4a5b-b59e-5342c92f8548" targetRef="END-N-d3219b76-e9c7-4a5b-b59e-5342c92f8548"/> <serviceTask id="N-d3219b76-e9c7-4a5b-b59e-5342c92f8548" name="微信" activiti:delegateExpression="#{ActionDelegateDispatcher}"> <extensionElements> <activiti:field name="parameterMap"> <activiti:string><![CDATA[{"triggerHistoryEvent":false,"channelId":"wx1f7fbd28ac009cd0","replyMessage":{"touser":"#{OPENID}","text":{"content":"近期活动 Upcoming events:\n\n2018/11/28 <a href=\" \" data-miniprogram-appid=\"wxf1d37b3137043a00\">报名 Register</a >\n荟同开讲 | 双语教育的模型与实践\nWhittle Talks | Bilingual Models and Practices\n\n2018/12/08 <a href=\"pages/order/order?event_id=82\" data-miniprogram-appid=\"wxf1d37b3137043a00\">报名 Register</a >\n荟同开讲 | 小童书的大说法\nWhittle Talks | The big story behind a book for small children\n\n更多精彩活动，请<a href=\"pages/calendar/calendar\" data-miniprogram-appid=\"wxf1d37b3137043a00\">查看活动日历</a >\n\nFind out more in our <a href=\"pages/calendar/calendar\" data-miniprogram-appid=\"wxf1d37b3137043a00\"> Event Calendar</a >"}},"resultVariable":"gatewayc58"}]]></activiti:string> </activiti:field> </extensionElements> </serviceTask> <sequenceFlow id="N-7f0dcb94-be72-4999-84dc-256fa65d9fd9" sourceRef="N-8f2e9577-46e5-4833-8cb6-e0622d531496" targetRef="N-d3219b76-e9c7-4a5b-b59e-5342c92f8548"/> </process> <bpmndi:BPMNDiagram id="BPMNDiagram_P9fbf8335-9f0a-4c8e-a1bf-a6c4ec856c1a"> <bpmndi:BPMNPlane bpmnElement="P9fbf8335-9f0a-4c8e-a1bf-a6c4ec856c1a" id="BPMNPlane_P9fbf8335-9f0a-4c8e-a1bf-a6c4ec856c1a"/> </bpmndi:BPMNDiagram> </definitions>
```
