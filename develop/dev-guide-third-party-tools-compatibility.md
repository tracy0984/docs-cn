---
title: 已知的第三方工具兼容问题
summary: 介绍在测试中发现的 TiDB 与第三方工具的兼容性问题。
---

# 已知的第三方工具兼容问题

> **注意：**
>
> TiDB 已列举[不支持的功能特性](/mysql-compatibility.md#不支持的功能特性)，典型的不支持特性有：
>
> - 存储过程与函数
> - 触发器
> - 事件
> - 自定义函数
> - 外键约束
> - 空间类型的函数、数据类型和索引
> - `XA` 语法
>
> 这些不支持的功能不兼容将被视为预期行为，不再重复叙述。关于更多 TiDB 与 MySQL 的兼容性对比，你可以查看[与 MySQL 兼容性对比](/mysql-compatibility.md)。

## 通用

### TiDB 中 `SELECT CONNECTION_ID()` 返回结果类型为 64 位整型

**描述**

TiDB 中 `SELECT CONNECTION_ID()` 的返回值为 64 位，如 `2199023260887`。而 MySQL 中返回值为 32 位，如 `391650`。

**规避方法**

在 TiDB 应用程序中，请注意使用各语言的 64 位整型类型（或字符串类型）存储 `SELECT CONNECTION_ID()` 的结果，防止溢出。如 Java 应使用 `Long` 或 `String` 进行接收，JavaScript/TypeScript 应使用 `string` 类型进行接收。

### TiDB 未设置 `Com_*` 计数器

**描述**

MySQL 维护了一系列 [`Com_` 开头的服务端变量](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html#statvar_Com_xxx)来记录你对数据库的操作总数，如 `Com_select` 记录了 MySQL 数据库从上次启动开始，总共发起的 `SELECT` 语句数（即使语句并未成功执行）。而 TiDB 并未维护此变量。你可以使用语句 [`SHOW GLOBAL STATUS LIKE 'Com_%'`](/sql-statements/sql-statement-show-status.md) 观察 TiDB 与 MySQL 的差异。

**规避方法**

请勿使用这样的变量。在 MySQL 中 `Com_*` 常见的使用场景之一是监控。TiDB 的可观测性较为完善，无需从服务端变量进行查询。如需定制监控工具，可阅读 [TiDB 监控框架概述](/tidb-monitoring-framework.md)来获得更多信息。

### TiDB 错误日志区分 `TIMESTAMP` 与 `DATETIME` 类型

**描述**

TiDB 错误日志区分 `TIMESTAMP` 与 `DATETIME`，而 MySQL 不区分，全部返回为 `DATETIME`。即 MySQL 会将 `TIMESTAMP` 类型的报错信息错误地写为 `DATETIME` 类型。

**规避方法**

请勿使用错误日志进行字符串匹配，要使用[错误码](/error-codes.md)进行故障诊断。

### TiDB 不支持 `CHECK TABLE` 语句

**描述**

TiDB 不支持 `CHECK TABLE` 语句。

**规避方法**

在 TiDB 中使用 [`ADMIN CHECK [TABLE|INDEX]`](/sql-statements/sql-statement-admin-check-table-index.md) 语句进行表中数据和对应索引的一致性校验。

## 与 MySQL JDBC 的兼容性

测试版本为 MySQL Connector/J 8.0.29。

### 默认排序规则不一致

**描述**

MySQL Connector/J 的排序规则保存在客户端内，通过获取的服务端版本进行判别。

下表列出了已知的客户端与服务端排序规则不一致的字符集：

| 字符集 | 客户端默认排序规则 | 服务端默认排序规则 |
| --------- | -------------------- | ------------- |
| `ascii`   | `ascii_general_ci`   | `ascii_bin`   |
| `latin1`  | `latin1_swedish_ci`  | `latin1_bin`  |
| `utf8mb4` | `utf8mb4_0900_ai_ci` | `utf8mb4_bin` |

**规避方法**

在 TiDB 中手动设置排序规则，不要依赖客户端默认排序规则。客户端默认排序规则由 MySQL Connector/J 配置文件保存。

### 参数 `NO_BACKSLASH_ESCAPES` 不生效

**描述**

TiDB 中 无法使用 `NO_BACKSLASH_ESCAPES` 参数从而不进行 `\` 字符的转义。已提 [issue](https://github.com/pingcap/tidb/issues/35302)。

**规避方法**

在 TiDB 中不搭配使用 `NO_BACKSLASH_ESCAPES` 与 `\`，而是使用 `\\` 编写 SQL 语句。

### 未设置索引使用情况参数

**描述**

TiDB 在通讯协议中未设置 `SERVER_QUERY_NO_GOOD_INDEX_USED` 与 `SERVER_QUERY_NO_INDEX_USED` 参数。这将导致以下参数返回与实际不一致：

- `com.mysql.cj.protocol.ServerSession.noIndexUsed()`
- `com.mysql.cj.protocol.ServerSession.noGoodIndexUsed()`

**规避方法**

不使用 `noIndexUsed()` 与 `noGoodIndexUsed()` 函数。

### 不支持 `enablePacketDebug` 参数

**描述**

TiDB 不支持 [enablePacketDebug](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-debugging-profiling.html) 参数，这是一个 MySQL Connector/J 用于调试的参数，将保留数据包的 Buffer。这将导致连接的**意外关闭**，**请勿**打开。

**规避方法**

不设置 `enablePacketDebug` 参数。

### 不支持 UpdatableResultSet

**描述**

TiDB 暂不支持 `UpdatableResultSet`，即请勿指定 `ResultSet.CONCUR_UPDATABLE` 参数，也不要在 `ResultSet` 内部进行数据更新。

**规避方法**

使用 `UPDATE` 语句进行数据更新，可使用事务保证数据一致性。

## MySQL JDBC Bug

### `useLocalTransactionState` 和 `rewriteBatchedStatements` 同时开启将导致事务无法提交或回滚

**描述**

`useLocalTransactionState` 和 `rewriteBatchedStatements` 两参数同时开启时，将导致事务无法提交。你可以使用[代码](https://github.com/Icemap/tidb-java-gitpod/tree/reproduction-local-transaction-state-txn-error)复现。

**规避方法**

> **注意：**
>
> 已向 MySQL JDBC 报告此 Bug，可关注此 [Bug Report](https://bugs.mysql.com/bug.php?id=108643) 进行最新消息的跟踪。

请勿开启 `useLocalTransactionState`，这有可能导致事务无法提交或回滚。

### Connector 无法兼容 5.7.5 版本以下的服务端

**描述**

MySQL Connector/J 8.0.29 在与 5.7.5 版本以下的 MySQL 服务端，或使用 5.7.5 版本以下 MySQL 服务端协议的数据库（如 TiDB 6.3.0 版本以下）同时使用时，将在某些情况下导致数据库连接的挂起。关于更多细节信息，可查看此 [Bug Report](https://bugs.mysql.com/bug.php?id=106252)。

**规避方法**

这是一个已知的问题，截至 2022 年 10 月 12 日，MySQL Connector/J 未合并修复代码。

TiDB 对其进行了两个维度的修复：

- 客户端方面：**pingcap/mysql-connector-j** 中修复了该 Bug，你可以使用 [pingcap/mysql-connector-j](https://github.com/pingcap/mysql-connector-j) 替换官方的 MySQL Connector/J。
- 服务端方面：TiDB v6.3.0 修复了此兼容性问题，你可以升级服务端至 v6.3.0 或以上版本。
