# 逻辑类型 → 各物理库类型

[field-conventions.md](field-conventions.md) 统一字段表给的是**逻辑类型**（`boolean` / `tinyint` / `bigint` / `datetime` / `int` / `varchar(n)`），与具体数据库无关。本文件是逻辑类型落到各物理库的对照表，**建表 / 写迁移脚本前先来这里换算**。

**只管统一字段**（`id` / `created_at` / `removed` 等）。业务字段用什么类型由项目按库选，本规范不约束（见 [SKILL.md 的作用范围](../SKILL.md)）。

## 1. 逻辑类型 → 物理类型

| 逻辑类型 | MySQL | PostgreSQL | Oracle | SQL Server | 达梦 DM | H2 |
|---|---|---|---|---|---|---|
| `boolean` | `TINYINT` | `SMALLINT` | `NUMBER(1)` | `TINYINT` | `TINYINT` | `TINYINT` |
| `tinyint` | `TINYINT` | `SMALLINT` | `NUMBER(3)` | `TINYINT` | `TINYINT` | `TINYINT` |
| `int` | `INT` | `INTEGER` | `NUMBER(10)` | `INT` | `INT` | `INT` |
| `bigint` | `BIGINT` | `BIGINT` | `NUMBER(19)` | `BIGINT` | `BIGINT` | `BIGINT` |
| `datetime` | `DATETIME` | `TIMESTAMP` | `TIMESTAMP` | `DATETIME2` | `TIMESTAMP` | `TIMESTAMP` |
| `varchar(n)` | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR2(n)` | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR(n)` |

> **`TINYINT` 不是通用类型，禁止跨库照抄**：
> - **PostgreSQL 没有 `TINYINT`** → 用 `SMALLINT`
> - **Oracle 没有 `TINYINT`** → 用 `NUMBER(3)`（`SMALLINT` 在 Oracle 只是 `NUMBER(38)` 的别名，太宽，不要用）
> - **SQL Server 的 `TINYINT` 是无符号 0–255**，存不了负数；需要 -1 之类哨兵值时改用 `SMALLINT`
> - 达梦 / H2 / MySQL 有原生 `TINYINT`
>
> 存 0/1 或小状态码的字段，逻辑类型写 `tinyint`，按上表换算。

> 上表之外的库（GaussDB、OceanBase、openGauss、KingbaseES…）按各自方言查手册换算，**不要照抄 MySQL 列**。

## 2. boolean 一律存 0/1

**所有逻辑 `boolean` 字段（`removed` 及各类状态位）一律落成 0/1 小整数，不使用库的原生 `BOOLEAN`。** 各库物理类型见 §1 的 `boolean` 行：

| MySQL | PostgreSQL | Oracle | SQL Server | 达梦 DM | H2 |
|---|---|---|---|---|---|
| `TINYINT` | `SMALLINT` | `NUMBER(1)` | `TINYINT` | `TINYINT` | `TINYINT` |

**取值与字面量全项目统一**：`0` = false（未删除 / 未启用），`1` = true（已删除 / 已启用）。DML 一律写整数，`removed = 0` / `removed = 1`，**不写 `FALSE` / `TRUE`**。

```sql
-- ✓ 所有库通用
UPDATE wo_workorder SET removed = 1 WHERE id = 1001;
SELECT * FROM wo_workorder WHERE removed = 0;
```

**为什么定死 0/1 而不是原生 `BOOLEAN`**：

- 原生 `BOOLEAN` 各库行为不一致——PostgreSQL 是 `BOOLEAN`，MySQL 是 `TINYINT`，SQL Server 是 `BIT`，Oracle 21c 干脆没有
- 字面量跟着分叉：PostgreSQL 的 `BOOLEAN` 列上写 `removed = 0` 直接报 `operator does not exist: boolean = integer`，测试库 H2 默认模式同样不兼容整数比较
- ORM 配置也得跟着分叉。MyBatis-Plus 的 `@TableLogic(value = "0", delval = "1")` 在 `BOOLEAN` 列上生成的 SQL 同样报上面的错
- 0/1 小整数各库都有，DDL / DML / 迁移脚本 / ORM 配置**零差异**

代价是语义弱一点，靠字段名 + `NOT NULL DEFAULT 0` 表达约束；需要硬约束时加 `CHECK (removed IN (0, 1))`。

**例外：历史表已经是原生 `BOOLEAN`**。改造成本高就保留，但**该表内的字面量必须跟它的物理类型走**（`BOOLEAN` 列写 `FALSE` / `TRUE`），ORM 的 `@TableLogic` 也按该表单独配，不要全局改成 0/1。

### ORM 侧必须让 Boolean 映射到 0/1，否则类型检查报错

Java 字段用 `Boolean` **却不改 ORM 的默认映射**时，schema 校验期望的列类型与本规范的 0/1 列对不上，`hbm2ddl.auto=validate`（或 `ddl-auto: validate`）直接失败：

| Hibernate 的 `Boolean` 默认期望 | 本规范的 0/1 列 | 报错 |
|---|---|---|
| MySQL → `bit` | `TINYINT` | `found [tinyint], but expecting [bit]` |
| PG / H2 → `boolean` | `SMALLINT` / `TINYINT` | `found [smallint], but expecting [boolean]` |
| SQL Server → `bit` | `TINYINT` | 类型不匹配 |
| Oracle → `number(1)` | `NUMBER(1)` | 通过 |

**只跑 MySQL 的项目一开始没事，一接 PostgreSQL、测试库 H2 或新开 Flyway 迁移就炸**——和 §2 的字面量问题同源。

**解法 A（推荐）：全局配置，一次搞定。** 把 Hibernate 的 `Boolean` 显式指向 `TINYINT`，让校验期望与 Flyway 写出的列一致：

```yaml
# application.yml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        type:
          preferred_boolean_jdbc_type: TINYINT
```

- 对应 Hibernate 配置项 `hibernate.type.preferred_boolean_jdbc_type`
- **全局生效，所有 `Boolean` 字段一律落 0/1**，正好等于本规范的要求
- PostgreSQL 没有 `TINYINT`，由方言落到 `SMALLINT`，与 §1 一致；**接入新库时验证一次实际生成的列类型**，对不上就按 §1 的映射手工写 DDL

**解法 B：逐字段声明转换**（无法改全局配置，或某几个字段要跟全局不同时用）。Hibernate 6+ 内置 `org.hibernate.type.NumericBooleanConverter`（`Boolean` ↔ `1`/`0`）：

```java
// 普通布尔字段
@Convert(converter = org.hibernate.type.NumericBooleanConverter.class)
private Boolean removed;

// 逻辑删除：Hibernate 6.5+ 的 @SoftDelete，官方文档示例的列名就叫 removed
@SoftDelete(columnName = "removed", converter = org.hibernate.type.NumericBooleanConverter.class)
private Boolean removed;
```

**解法 C：Java 字段直接用 `Integer` / `Short`**，与列类型完全一致、零配置，代价是丢掉 `Boolean` 语义（判断要写 `removed == 0`）。

**MyBatis-Plus 不受影响**：`@TableLogic(value = "0", delval = "1")` 配 `Boolean` 字段即可，`BooleanTypeHandler` 在 JDBC 层转换，也不做 schema 校验。

> Entity 字段映射注解的完整清单、`@SoftDelete` 的策略选项 → 后端框架类 skill。本节只覆盖「本规范定了 0/1 存储，ORM 侧必须配什么」。

## 3. id：类型取决于 ID 生成策略

`id` 没有唯一固定类型，**先定生成策略再定类型**：

| ID 生成策略 | 逻辑类型 | 物理类型 | 说明 |
|---|---|---|---|
| 数据库自增 | `bigint` | 见 §1 的 `bigint` 行 | 由库生成，最省事，但分库分表 / 数据迁移时会撞 |
| 雪花 ID（数值） | `bigint` | 见 §1 的 `bigint` 行 | 64 位整数，应用侧生成 |
| 雪花 ID（字符串）/ ULID / UUID | `varchar(36)` | MySQL/PG/SQL Server/达梦/H2 `VARCHAR(36)`，Oracle `VARCHAR2(36)` | 应用侧生成，全链路当字符串 |

- `varchar(36)` 是**上限**：UUID 带连字符 36 字符、不带 32、ULID 26，都装得下；不要按各自长度分表设计
- **为什么雪花 ID 有时用字符串存**：数值型雪花 ID 超过 2^53，JSON / JS 侧会丢精度，所以全链路（DB → Java → 报文 → 前端）统一按字符串走。是否这么做由 API 契约定，本规范只管**一旦定了字符串，DB 就用 `varchar(36)`，不要一半表 `bigint` 一半表 `varchar`**
- **关联字段必须与被引用主键同类型**——`id` 改成 `varchar(36)` 时，所有 `<引用表功能点>_id` 跟着改，见 [foreign-keys.md](foreign-keys.md)
- 用雪花 / ULID / UUID 时**不要**加自增 / `IDENTITY`，主键由应用生成

### 自增主键的 DDL 写法（各库不同）

| 库 | 写法 |
|---|---|
| MySQL | `id BIGINT NOT NULL AUTO_INCREMENT` |
| PostgreSQL | `id BIGINT GENERATED BY DEFAULT AS IDENTITY`（或 `BIGSERIAL`） |
| Oracle | `id NUMBER(19) GENERATED BY DEFAULT AS IDENTITY`（12c+；11c 用 SEQUENCE + 触发器） |
| SQL Server | `id BIGINT IDENTITY(1,1)` |
| 达梦 DM | `id BIGINT IDENTITY(1,1)` |
| H2 | `id BIGINT GENERATED BY DEFAULT AS IDENTITY` |

## 4. 统一字段 DDL 对照

同一张表在三个库的落法（自增 `id`，`removed` 存 0/1）：

```sql
-- MySQL
CREATE TABLE wo_workorder
(
  id BIGINT NOT NULL AUTO_INCREMENT,
  created_at DATETIME NOT NULL,
  created_by BIGINT NOT NULL,
  removed TINYINT NOT NULL DEFAULT 0
);

-- PostgreSQL
CREATE TABLE wo_workorder
(
  id BIGINT GENERATED BY DEFAULT AS IDENTITY,
  created_at TIMESTAMP NOT NULL,
  created_by BIGINT NOT NULL,
  removed SMALLINT NOT NULL DEFAULT 0
);

-- Oracle
CREATE TABLE wo_workorder
(
  id NUMBER(19) GENERATED BY DEFAULT AS IDENTITY,
  created_at TIMESTAMP NOT NULL,
  created_by NUMBER(19) NOT NULL,
  removed NUMBER(1) NOT NULL DEFAULT 0
);
```

`id` 换成 ULID / UUID 时：类型改成 `VARCHAR(36)` / `VARCHAR2(36)`，去掉 `AUTO_INCREMENT` / `IDENTITY`，`created_by` 等关联字段跟着同类型。

排版（关键字大写、2 空格缩进、不等号对齐等）见 [sql-format.md](sql-format.md)，本文件只管类型怎么换算。
