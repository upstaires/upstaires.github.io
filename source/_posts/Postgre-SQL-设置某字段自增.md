---
title: Postgre SQL 设置某字段自增
author: upstairs
tags:
  - Postgre SQL  # 具体技术标签
categories:
  - 数据库  # 必须指定有效分类
abbrlink: 57705
date: 2025-06-21 10:45:22
---

# Postgre SQL 设置某字段自增

在PostgreSQL中，假设通过Navicate从别的数据库拷贝数据表到Postgres数据库，原本有一个字段是自增的，现在不是自增的，需要设置为自增。

新建一个序列,名字随意，能记住就行。
``` sql
Create sequence seq_id;
SELECT setval('seq_id', COALESCE((SELECT MAX(id) FROM table_name), 0), true);
```
注：这里的id是需要自增的的字段名，table_name是需要设置自增的表名。

然后在设计表中的默认栏填写:
``` sql
nextval('seq_id'::regclass)
```
使用效果是插入数据后，该字段自增，但是你删除了一条数据，在插入，此时自增字段的值为之前的最大值+1，也就是说如果你把自增字段最大值13删除，在此插入一个，此时字段值是14，不是13，因为13已经被用过了。