---
title: 分布式事务
date: 2021-09-01 16:57:57
tags:
  - 微服务
  - 分布式事务
---

# 使用场景

## 转账

支付宝账户表：A(id, user_id, amount)
余额宝账户表：B(id, user_id, amount)

user_id = 1 的用户，从支付宝转账 1 万元到余额宝分为两个步骤：

1. 支付宝账户表扣款 1 万元
2. 余额宝账户表增加 1 万元

在 MySQL 中，可以通过事务保证一致性

```mysql
begin;
update A set amount=amount-10000 where user_id=1;
update B set amount=amount+10000 where user_id=1;
commit;
```

随着系统变大，我们进行了微服务的改造，每个微服务独占一个数据库。这时转账动作跨越了两个微服务：pay 和 balance。如何保证跨服务的事物一致性？


# 解决方案

## 事务消息

首先系统应该保证每个服务自身的 ACID，基于此，我们可以通过事务消息来保证一致性。

在扣款的同时创建一条消息，消息里记录“让 user_id=1 的余额宝账户增加 1 万元”。我们通过这个凭证保证最终一致性。

## Best Effort

## Transactional outbox

## Polling publisher

## Transaction log tailing

## 幂等

## 2PC

## 2PC Message Queue

## Seata 2PC

## TCC