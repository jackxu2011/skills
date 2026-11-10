# 统一字段

建表时需要审计字段、乐观锁、逻辑删除、排序字段 → 读这一份。**对象命名（表/视图/索引前缀）在 [naming.md](naming.md)，本文件只管字段。**

**核心原则：字段名固定，是否使用看场景。** 如果某张表需要这些字段，**必须**采用下列命名，不要自创 `create_time` / `modify_user` / `is_deleted` 等变体。

下表的类型是**逻辑类型**（`boolean` / `tinyint` / `bigint` / `datetime` / `int` / `varchar`），与具体数据库无关。落到 MySQL / PostgreSQL / Oracle / SQL Server / 达梦 / H2 的物理类型、DML 字面量写法，见 [type-mapping.md](type-mapping.md)——**建表前先去那里换算，不要把下表的逻辑类型直接抄进 DDL**。

两个字段的类型**不是固定的**，必须先定策略再落类型：
- **`id`** —— 自增 / 雪花数值是 `bigint`，ULID / UUID / 字符串型雪花是 `varchar(36)`，**同一批表要统一**
- **`removed`** —— 逻辑 `boolean`，**一律存 0/1**（0=未删除 / 1=已删除），不使用库的原生 `BOOLEAN`

两者都见 [type-mapping.md](type-mapping.md)。

## 1. 通用字段（按需选用）

| 字段 | 逻辑类型 | 用途 | 何时**不**需要 |
|------|---------|------|--------------|
| `id` | `bigint` / `varchar(36)`，**按 ID 生成策略定** → [type-mapping.md](type-mapping.md) | 主键 | 无主键的临时表/中间表 |
| `created_at` | `datetime` | 创建时间 | 静态字典表 |
| `created_by` | `bigint` / `varchar` | 创建人 | 无登录场景（公开提交）、系统自动产生的记录 |
| `updated_at` | `datetime` | 修改时间 | 记录一旦创建不可修改（流水/日志/审计） |
| `updated_by` | `bigint` / `varchar` | 修改人 | 同上 |
| `removed` | `boolean` 存 0/1 | 逻辑删除标记，0=未删除 / 1=已删除 | 不需要软删除（物理删除即可，或记录永久保留） |

## 2. 扩展字段（按需添加，命名固定）

| 字段 | 用途 | 何时使用 |
|------|------|---------|
| `sort_order` | 排序权重，逻辑类型 `int`，默认 0，升序 | 列表需要人工/规则排序时 |
| `version` | 乐观锁版本号，逻辑类型 `int`，默认 0，每次 UPDATE 自增 1 | 并发更新场景 |
| `tenant_id` | 租户隔离 | 多租户 SaaS |
| `remark` | 备注 | 业务允许自由文本说明 |
| `status` | 业务状态 | 用 `<语义>_status` 更佳，如 `order_status`，避免单独 `status` 在多表间混淆 |

## 3. 典型场景示例

- **业务实体表**（订单、客户、工单）：`id` + `created_at` + `created_by` + `updated_at` + `updated_by` + `removed` 全套
- **登录日志/审计流水**：`id` + `created_at`，无 `updated_*`（不可修改）、无 `removed`（不删除）、无 `created_by`（操作人由业务字段 `user_id` / `operator_id` 表达，避免与元数据重复）
- **公开表单提交**（如匿名反馈）：`id` + `created_at`，无 `created_by`（无登录）
- **字典表**（数据字典、枚举）：`id` + `created_at` + `updated_at`，无 `*_by`（系统初始化）

## 4. 命名约定（用到这些字段时必须遵守）

- 时间字段统一 `_at` 后缀（`created_at` / `updated_at` / `published_at`），不要用 `create_time` / `update_time`
- 人员字段统一 `_by` 后缀（`created_by` / `updated_by` / `approved_by`）
- 软删除字段固定叫 `removed`，`boolean` 存 0/1（0=未删除 / 1=已删除），DML 一律写 `removed = 0` / `removed = 1`。不要用 `is_deleted` / `deleted` / `del_flag`，也**不要**用库的原生 `BOOLEAN`。各库物理类型见 [type-mapping.md](type-mapping.md)
- 乐观锁字段固定叫 `version`，逻辑类型 `int` 默认 0，不要用 `ver` / `revision` / `lock_version` / `opt_lock`（MyBatis-Plus / JPA `@Version` 的事实约定）。如果表本身有**业务版本**语义（文档版本、模板版本等），业务版本改名 `revision` / `<entity>_version`（如 `doc_version`、`template_version`），`version` 留给乐观锁
- 排序字段固定叫 `sort_order`，逻辑类型 `int` 默认 0，升序，不要用 `order` / `sort` / `seq` / `display_order`（`order` 是 SQL 关键字，`sort` 在部分 ORM/方言里也有歧义）
- 布尔/状态标记统一 `boolean` **存 0/1**（`TINYINT` / `SMALLINT` / `NUMBER(1)`），**不要**用库的原生 `BOOLEAN`，也**不要**用 MySQL `ENUM` 类型（修改困难、移植性差）。ORM 侧必须显式声明 0/1 转换，否则 Hibernate 类型检查报错 → [type-mapping.md](type-mapping.md)

> **注**：如果项目有特殊规定（如历史表已用 `del_flag`、业务方约定字段命名等），按项目规定走；新表/新字段遵循上述规范。

## 相关

- `removed` 与唯一索引冲突、自动填充不触发等运行时问题 → 运行时陷阱排查类 skill（MySQL / MyBatis-Plus 分类）
- Entity 侧这些字段怎么映射（`@TableField`、逻辑删除、乐观锁注解）→ 后端框架类 skill
