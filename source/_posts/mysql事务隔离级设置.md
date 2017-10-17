---
title: mysql事务隔离级设置
date: 2017-08-24 11:34:09
tags: mysql 事务命令
---

## mysql 事务隔离级设置

### 如何查看当前SESSION的事务隔离级
mysql 默认的事务隔离级是 REPEATABLE-READ

可以通过以下命令查看当前session的事务隔离级（5.7.20之前）
```
MariaDB [(none)]> select @@session.tx_isolation;
+------------------------+
| @@session.tx_isolation |
+------------------------+
| REPEATABLE-READ        |
+------------------------+
1 row in set (0.00 sec)
```

### 如何修改事务隔离级
#### 参考 [mysql手册](https://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)
```
SET [GLOBAL | SESSION] TRANSACTION
    transaction_characteristic [, transaction_characteristic] ...

transaction_characteristic:
    ISOLATION LEVEL level
  | READ WRITE
  | READ ONLY

level:
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
```

1. GLOBAL:代表对随后所有的SESSION都生效，已经建立的SESSION不受影响
2. SESSION: 代表只对当前的SESSION生效

