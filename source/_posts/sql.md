---
title: insert和replace相关
date: 2019-03-20
tags:
---
### insert ignore into 和insert into
INSERT IGNORE 与INSERT INTO的区别就是INSERT IGNORE会忽略数据库中已经存在 的数据，如果数据库没有数据，就插入新的数据，如果有数据的话就跳过这条数据。这样就可以保留数据库中已经存在数据，达到在间隙中插入数据的目的。


### replace into 
replace into表示插入替换数据，需求表中有PrimaryKey，或者unique索引的话，如果数据库已经存在数据，则用新数据替换，如果没有数据效果则和insert into一样；
REPLACE语句会返回一个数，来指示受影响的行的数目。该数是被删除和被插入的行数的和。如果对于一个单行REPLACE该数为1，则一行被插入，同时没有行被删除。如果该数大于1，则在新行被插入前，有一个或多个旧行被删除。如果表包含多个唯一索引，并且新行复制了在不同的唯一索引中的不同旧行的值，则有可能是一个单一行替换了多个旧行。


### insert into ... on duplicate key update  
insert语句的末尾添加on duplicate key update语法：如果插入行出现**唯一索引或者主键**重复时，则执行旧的update；如果不会导致唯一索引或者主键重复时，就直接添加新行。
https://blog.csdn.net/qq_41070393/article/details/82422632
