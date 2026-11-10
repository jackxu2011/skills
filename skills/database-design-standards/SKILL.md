---
name: database-design-standards
description: 数据库设计规范：库/表/字段/索引/约束命名、SQL 脚本格式化（关键字大写/缩进）、表结构设计（统一字段、默认不建物理外键）、字符集、数仓分层。Use when 写或评审 DDL/SQL（含 MyBatis XML、Flyway 迁移脚本）、设计表结构、命名表/字段/索引。
metadata:
  version: "1.0.0"
---

# 数据库设计规范 (Database Design Standards)

**作用范围**：本规范管命名、SQL 格式、字符集、统一字段、外键约束策略、数仓分层；**不管**业务字段的类型选择（VARCHAR/TEXT/BIGINT 等）和数据建模（范式/分库分表）。

**唯一例外**：统一字段（`id` / `created_at` / `removed` / `version` 等）必须全库一致，所以它们的**逻辑类型 + 各物理库映射**在本规范内 → [references/type-mapping.md](references/type-mapping.md)。业务字段的类型仍然不管。

SKILL.md 只放**硬约束 + 取名 5 步 + 自检清单 + 索引**，细则按主题拆在 `references/` 下。**只读当前任务需要的那一个文件。**

## 硬约束（最容易被静默违反，先记住再动手）

- 表名 = `<模块前缀>_<功能点>`，全小写、**单数**、≤ 64 字符（**建议 ≤ 30**，给索引名留余量）、**不带库前缀**（`wo_workorder` ✓，`app_wo_workorder` ✗）
- 用完整英文单词或公认缩写，**禁止拼音及拼音-英文混合**（`wo_gongdan`、`cu_kehu` ✗）
- 模块前缀 2-4 字母，常用 `wo` `cu` `fin` `sys` `org` `prod` `ord` `inv` `log` `all`；不得占用对象前缀（`fn`/`i`/`seq`/`pk`…全表见 naming.md）
- 索引 `i_<表名>_<字段>`、唯一键 `uk_<表名>_<字段>`、主键 `pk_<表名>`；**表名段不许省**（`i_status` ✗）——**索引名唯一范围各库不同**（PG / Oracle 是 **schema 内唯一**，MySQL / SQL Server / H2 是表内唯一），省了表名段换库就撞名
- 索引/约束名 ≤ 目标库上限（MySQL 64、PgSQL 63 字节且**静默截断**）：超长时**先缩词后砍段，缩词先缩表名段**（表名段最长、是最大来源），仍超再缩字段名段；**每个单词缩到 2–4 字符**（`customer`→`cst`、`reconciliation`→`rcn`、`status`→`sts`），极端时**多词首字母连写**（`customer_reconciliation`→`cr`），砍掉整列是最后手段；禁省表名段、禁随机后缀 → [naming.md 降级阶梯](references/naming.md#对象名长度与超长降级)
- 用到统一字段就叫对名字：`created_at` / `created_by` / `updated_*` / `removed` / `version` / `sort_order`——禁 `create_time` / `is_deleted` / `del_flag` 变体
- **默认不建物理外键**：不写 `FOREIGN KEY` / `REFERENCES` / `ON DELETE CASCADE`；关联字段**必须被索引覆盖**——单独 `i_<表名>_<字段>`，**或**联合索引的**最左前导列**（只落在联合索引的非前导位置不算覆盖）
- 统一字段在规范里只给**逻辑类型**（`boolean`/`tinyint`/`bigint`/`datetime`/`int`），DDL 必须落成**目标物理库**的类型——**`TINYINT` 不是通用类型**（PG 用 `SMALLINT`、Oracle 用 `NUMBER(3)`、SQL Server 的 `TINYINT` 是无符号 0–255），**禁止跨库照抄 MySQL** → [references/type-mapping.md](references/type-mapping.md)
- **`id` 的类型先定生成策略再落**：自增 / 雪花数值是 `bigint`，ULID / UUID / 字符串型雪花是 `varchar(36)`，同批表必须统一；用雪花 / ULID / UUID 时不加自增 → [references/type-mapping.md](references/type-mapping.md)
- **`boolean`（`removed` 及各类状态位）一律存 0/1**（0=未删除 / 未启用，1=已删除 / 已启用），**不用库的原生 `BOOLEAN`**；DML 一律写 `removed = 0` / `removed = 1`。ORM 侧必须让 `Boolean` 映射到 0/1（`hibernate.type.preferred_boolean_jdbc_type: TINYINT`），否则 schema 校验报 `found [tinyint], but expecting [bit]` → [references/type-mapping.md](references/type-mapping.md)
- SQL 文本：关键字大写、2 空格缩进、禁 `SELECT *`、`INSERT` 必须指定字段名
- 字符集统一 UTF-8，MySQL 用 `utf8mb4`

## Quick Start：命名一张表的 5 步

> 实体类名 = 表名去掉模块前缀，所以**表名设计直接决定 Java 类长什么样**。

1. **选模块前缀**（2-4 小写字母）—— 见 [naming.md 的常用模块前缀表](references/naming.md#数据库对象命名规则)。不得与对象前缀冲突（`fn`、`i`、`seq`、`pk` 等已占用）
2. **取功能点**（≤ 3 个完整英文单词，单数，业务名词或动宾）—— 必须**自解释**：去掉前缀后能让同事一眼看懂
3. **拼接**：`<模块前缀>_<功能点>`，全小写，下划线连接，**不带库前缀**（`wo_workorder` ✓，`app_wo_workorder` ✗）
4. **检查长度**：硬上限 MySQL ≤ 64 字符，**建议 ≤ 30**——索引名/外键名要在表名之上再加前缀和字段名，表名吃满 64 就没余量了（见 [naming.md 的长度预算](references/naming.md#对象名长度与超长降级)）
5. **按场景选用统一字段**：常见组合 = `id` + `created_at` + `created_by` + `updated_at` + `updated_by` + `removed`；流水/日志类省略 `updated_*`、`removed`；匿名提交省略 `_by` 字段——**命名固定，但是否使用看场景**，细则见 [field-conventions.md](references/field-conventions.md)

**示例**：工单模块下"工单分配人员表" → 前缀 `wo` + 功能点 `assign_user` → `wo_assign_user` → 生成实体 `AssignUser.java` ✓

## 产出前自检

写完 DDL / SQL 逐条对照，任一条不过就改完再从头查一遍：

- [ ] 表名 = `<模块前缀>_<功能点>`，全小写、单数、≤ 64 字符、**不带库前缀**
- [ ] 功能点自解释，未用 `info` / `data` / `record` / `item` / `main` / `detail` / `log`
- [ ] 模块前缀未占用对象前缀（`fk` / `i` / `seq` / `pk` 等 19 个）
- [ ] 用到统一字段就叫对名字：`created_at` / `created_by` / `updated_*` / `removed` / `version` / `sort_order`，无 `create_time` / `is_deleted` / `del_flag` 变体
- [ ] 每个关联字段都被索引覆盖：单独 `i_<表名>_<字段>`，**或**在某个联合索引里当**最左前导列**（只在非前导位置 = 没覆盖）——默认不建物理外键，索引是 JOIN 性能的唯一保障
- [ ] 索引/约束名未超目标库上限（MySQL 64、PgSQL 63）：超长时**先缩表名段，再缩字段名段，最后才砍段**，每个单词**缩到 2–4 字符**且全项目一致，**表名段保留**、无随机后缀
- [ ] SQL 文本：关键字大写、2 空格缩进、无 `SELECT *`、`INSERT` 指定字段名
- [ ] 统一字段的类型已按**目标物理库**换算（不是照抄逻辑类型，**更不是照抄 MySQL 的 `TINYINT`**）→ [references/type-mapping.md](references/type-mapping.md)
- [ ] `id` 已按生成策略定类型且同批表一致（自增/雪花数值 `bigint`，ULID/UUID `varchar(36)`），关联字段 `<引用表功能点>_id` 与被引用主键同类型
- [ ] `boolean` 字段一律存 0/1，DML 写 `0`/`1` 而非 `FALSE`/`TRUE`；ORM 已配 `preferred_boolean_jdbc_type: TINYINT`（或逐字段 `@Convert(NumericBooleanConverter.class)`）
- [ ] 字符集 `utf8mb4`

## 索引

| 需要 | 加载文件 |
|------|----------|
| 19 种对象前缀全表、库名规则（`app_`/`ods_`/`ext_`）、数仓分层命名、表名/字段取名细则、模块前缀表、长度上限与超长降级阶梯 | [references/naming.md](references/naming.md) |
| 统一字段：通用/扩展字段表、哪些表省哪些字段、`_at` / `_by` / `removed` / `version` / `sort_order` 约定 | [references/field-conventions.md](references/field-conventions.md) |
| 统一字段的**逻辑类型**落到 MySQL / PostgreSQL / Oracle / SQL Server / 达梦 / H2 的物理类型、**`id` 按生成策略怎么定类型**（自增/雪花数值 `bigint` vs ULID/UUID `varchar(36)`）与各自自增 DDL 语法、**boolean 一律 0/1** 及 ORM 侧怎么配才不报类型错 | [references/type-mapping.md](references/type-mapping.md) |
| 要不要建外键、关联字段索引怎么算覆盖（单独 / 联合索引前导列）、物理外键的例外条件与理由 | [references/foreign-keys.md](references/foreign-keys.md) |
| SQL 缩进风格 / 空格换行 / CREATE TABLE 排版 / `SELECT *` 例外 / 字符集 | [references/sql-format.md](references/sql-format.md) |

## 与其它 skill 的边界

| 这个 skill | 留给别的 skill |
|---|---|
| 表/字段/索引怎么命名、DDL 怎么写 | 脚本文件放哪个目录、Flyway `V__` / `R__` 命名 → 项目目录与包结构类 skill |
| SQL 文本的书写格式 | Java 侧 ORM/Entity 写法 → Java 编码规范类 skill；Entity 字段映射注解 → 后端框架类 skill |
| 建表该建什么 | 索引失效、逻辑删除 + 唯一索引冲突等**运行时症状** → 运行时陷阱排查类 skill |
