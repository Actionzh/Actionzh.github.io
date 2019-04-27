---
title: flow(active)流程
date: 2019-03-29 10:01:16
tags:
---
active相关旅程。
CustomerJourneyController.activate方法：
1先保存customerJourney
2然后调用customerJourneyService.activate(triggerId)：
2.1查询出customerJourney，根据uimodel转换出LinkedJourney，
2.2再根据LinkedJourney得到相关得properties加入customerJourney
2.3根据customerJourney和linkedJourney进行deploy
```
deploy(
[1].bpmnGenerator.generateBPMN->converter出bpmnModel，[2].bpmnGenerator.getTriggerCondition(triggerNode)->根据triggerNode得到comditionstring，此处根据根节点workflowEvent得出beanname 从而得到对应得TriggerConditionGenerator接口得具体实现，核心方法generateInternal主要用来返回生成的condition字符串。(所有得trigger generator在此处使用)
[3].customerJourney设置得到的bpmn，triggerType，workflowEvent，condition
[4].根据customerJourneyname和bpmn进行deploy并返回ProcessDefinitionId,并设置到customerJourney中
[5].customerJourney中每个properties都设置processDedfId和triggerId)
```

2.4 deploy结束返回customerJourney，设置ACTIVE状态，并更新表数据
2.5最后保存所有properties节点信息

