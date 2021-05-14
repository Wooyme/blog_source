---
title: SQL简单调优(1)
date: 2018-07-30 11:53:35
tags:
    - SQL
---

## **1.前言**
先说一下背景，这几天在公司里接触了几个比较大的表，数据量在10万条到400万条之间，公司用的数据库是Oracle。所以如果这篇博客中描述的情况与你所遇到的情况不符的话，很有可能是数据量和数据库的问题。

## **2.联表与子查**
首先说一下业务的需求:*输入一个时间段，找到这个时间段里流失的客户。客户流失的逻辑:如果这个客户从最后一次下单到下单后的45天内没有下单则认为这个客户流失了*  数据库里有一张表(sap_vbak)维护着所有的订单，其中有订单号（vbeln),对应客户(kunnr),创建日期(erdat)。  
一开始拿到这个需求的时候想的很简单，大致分为两步:1.取出时间段内所有的订单 2.得到所有订单时间后计算出45天的时间段再取出这个时间段内所有的订单，如果不存在则认为客户流失，于是SQL语句就变成这样
```SQL
SELECT vbeln,kunnr,erdat FROM (
  SELECT k.vbeln,k.kunnr,k.erdat,(SELECT temp.vbeln FROM sap_vbak temp WHERE temp.erdat < k.erdat and temp.erdat >= k.erdat - 45 and temp.kunnr = k.kunnr and rownum = 1) tmp FROM sap_vbak k
  WHERE k.erdat >= to_date('2018-01-01','yyyy-MM-dd') and k.erdat <= to_date('2018-01-05','yyyy-MM-dd') group by k.vbeln,k.kunnr,k.erdat
) WHERE tmp is null;
```
三层的查询，逻辑非常清楚。但是这里问题就来了，由于数据量太大了，这个SQL要想跑出结果得花15分钟。其中对于速度影响最大的就是这里的 *标量子查询* 了。  
于是在公司前辈的指点下，把子查询改成了联表
```SQL
SELECT k.vbeln,k.kunnr,k.erdat FROM sap_vbak k
LEFT JOIN sap_vbak temp ON temp.erdat < k.erdat and temp.erdat >= k.erdat - 45 and temp.kunnr = k.kunnr
WHERE k.erdat >= to_date('2018-01-01','yyyy-MM-dd') and k.erdat <= to_date('2018-01-15','yyyy-MM-dd') and temp.vbeln is null;
```
这段SQL的效果非常好，跑出结果大概只要40秒。思路其实也很简单，把原先子查询的部分换成LEFT JOIN，得到一张临时表这张表记录着每个订单与它45天内同一个客户的所有订单，然后只要在这张表中选出tmp字段(即同一客户的订单)是null的结果即可。

后来发现另一种不通过联表的方法也可以提高查询效率
```SQL
SELECT k.vbeln,k.kunnr,k.erdat FROM sap_vbak k
WHERE k.erdat >= to_date('2018-01-01','yyyy-MM-dd') and k.erdat <= to_date('2018-01-05','yyyy-MM-dd')
and not exists (SELECT temp.vbeln FROM sap_vbak temp WHERE temp.erdat < k.erdat and temp.erdat >= k.erdat - 45 and temp.kunnr = k.kunnr);
```
把原先的标量子查询改成了WHERE后面的 *not exists* ，查询时间也从原本的15分钟，变到了5分钟以内。
