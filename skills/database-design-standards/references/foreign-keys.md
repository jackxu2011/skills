# 外键约束策略

要不要写 `FOREIGN KEY`、关联字段怎么建索引 → 读这一份。

**默认不建物理外键，关联关系由程序保证。**

- 建表语句**不写** `FOREIGN KEY` / `REFERENCES`，不用 `ON DELETE CASCADE` / `ON UPDATE CASCADE`
- 关联字段仍按 `<引用表功能点>_id` 命名（如 `wo_workorder.customer_id` 指向 `cu_customer.id`），字段类型与被引用主键保持一致
- **关联字段必须被索引覆盖**——没有外键约束时，这个索引是 JOIN 和按关联字段查询的唯一性能保障。两种覆盖方式，**任选其一即可，不要重复建**：
  1. 单独普通索引：`idx_<表名>_<字段>`（如 `idx_wo_workorder_customer_id`）
  2. 联合索引的**最左前导列**：`idx_wo_workorder_customer_id_removed`（`customer_id` 打头）已经覆盖了 `customer_id`，此时再单独建 `idx_wo_workorder_customer_id` 是纯负担（多占空间、拖慢写入，优化器还可能选错）
  - 唯一键 `uk_<表名>_<关联字段>`、以该字段打头的复合主键，同样算覆盖
  - **不算覆盖**：字段只出现在联合索引的**非前导位置**——`idx_wo_workorder_removed_customer_id` 里的 `customer_id` 按最左前缀走不上索引，等于没建。最左前缀的命中细节 → 运行时陷阱排查类 skill（MySQL 分类）
  - 关联字段在联合索引里排第几位按查询定：高频等值条件放前面，范围条件放最后
  - 索引名拼出来超目标库上限 → [naming.md 的降级阶梯](naming.md#对象名长度与超长降级)
- 引用完整性（存在性校验、级联删除/置空、孤儿数据清理）由 Service 层在事务内完成
- `fk_` 前缀仍保留在命名规范中，仅供确需物理外键的场景（见下）使用

## 理由

物理外键会带来分库分表/数据迁移障碍、写入时的额外锁与性能开销、批量导入与刷数据困难、报错信息对业务层不友好。

## 例外

需 DBA 或架构评审确认后才可使用物理外键：

- 强一致性要求极高且无分库分表规划的核心配置表
- 第三方系统或历史遗留库已有外键，改造成本高于收益

## 示例

> 下面示例用 MySQL 类型（`TINYINT` / `DATETIME`），只为演示索引覆盖关系；其它库的类型换算见 [type-mapping.md](type-mapping.md)。

```sql
-- ✓ 推荐 A：单独索引覆盖关联字段
CREATE TABLE wo_workorder
(
  id BIGINT NOT NULL,
  customer_id BIGINT NOT NULL,
  removed TINYINT NOT NULL DEFAULT 0,
  created_at DATETIME NULL
);
CREATE INDEX idx_wo_workorder_customer_id ON wo_workorder (customer_id);

-- ✓ 推荐 B：联合索引以关联字段打头 —— 同样算覆盖，不要再单独建 idx_wo_workorder_customer_id
CREATE INDEX idx_wo_workorder_customer_id_removed ON wo_workorder (customer_id, removed);

-- ✗ 不算覆盖：关联字段在非前导位置，按 customer_id 过滤或 JOIN 时走不上索引
CREATE INDEX idx_wo_workorder_removed_customer_id ON wo_workorder (removed, customer_id);

-- ✗ 禁止：物理外键约束
ALTER TABLE wo_workorder
  ADD CONSTRAINT fk_wo_workorder_cu_customer
  FOREIGN KEY (customer_id) REFERENCES cu_customer (id) ON DELETE CASCADE;
```

## 相关

- `fk_<本表>_<引用表>` 的命名方向、`idx_` / `uk_` 前缀全表 → [naming.md](naming.md#前缀命名规则)
- 建表语句排版（缩进、字段对齐）→ [sql-format.md](sql-format.md)
