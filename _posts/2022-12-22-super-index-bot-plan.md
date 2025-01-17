---
layout:       post
title:        "超级索引实现方案"
author:       "Hamza"
header-style: text
catalog:      true
tags:
    - bot
    - telegram
---

## 数据来源

- 已有索引机器人：如 @hao1234bot
- google高级搜索：选定 网站为t.me
- tg索引网站：如 <https://meow.tg/>

## 功能需求

- 支持搜索群组、频道、机器人，以下统称实体
- 支持实体分类：人为设定若干分类
- 实体的属性包括：名字、链接、id、类别、描述、tags、收录时间、来源（爬取/手动导入/用户收录)、状态（公有/私有/活跃/已注销/...）
- 支持收录功能：收录时可以指定实体属性
- 实体的状态定时更新，具体的，可以指定更新间隔，当用户查询时，超过该间隔的实体进行更新
- 查询结果可以显示实体的状态，用户可以在未点击链接前得知实体是否存在等信息
- 支持按热度排序功能：群组热度、关键词热度
- 广告系统：发布广告、竞价排名、自定义广告价格等

## 技术方案

- 采用go语言实现机器人后端，go语言对协程支持较好，相对于python能做到更高的并发
- 采用关系型数据库MYSQL存储数据
- 采用全文搜索引擎Elasticsearch搜索数据库中的关键词，返回相关结果
- 由于数据库中数据比较重要，考虑数据库双节点主从备份，或者采用ETCD或Redis实现分布式架构，既可以备份数据库，又可以做负载均衡，提高性能