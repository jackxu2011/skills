# 数据库命名规范详解

SKILL.md 是索引；需要 19 种对象前缀全表、库名规则、表名/字段取名细则、长度上限与超长降级阶梯、字符集合规写法、数仓分层命名时加载。**统一字段（`created_at` / `removed` / `version` 等）不在本文件，见 [field-conventions.md](field-conventions.md)。**

## 引言

### 文档目的

本文档为数据库开发设计规范文档，基于基本的数据库环境下，制定了一系列创建命名规范，实现对数据库数据流的统一规划和管理，从而合理利用资源成本，降低团队运维管理成本和学习成本，加快数据交付速度的目的。为后续项目实施开发提供基础稳定的平台。

### 适用范围

具体适用于各个产品线的数据平台搭建和初始化，产品线整体基于该基础平台之上建立系统，从而做到统一规划产品数据流问题。

### 整体方案

基于以上的目的和要求，实现各个产品线之间数据流的统一规划和管理，创建数据库，统一制定数据库初始化要求。

## 命名规范

### 基本命名原则

**原则1：在所有命名中，尽量使用标准、完整的英文单词。**

- 如果字段所表示的意义，其英文单词比较短，则尽量使用完整的英文单词。
- 如果单词较长则采用大家基本可以理解的缩写，如果没有公有的缩写，则可取单词的头几个字母形成缩写。
- **禁止拼音或拼音-英文混合**（`wo_gongdan`、`cu_kehu`、`fin_chuli_record` 一律不允许），缩写只在"大家都认"的范围内取。
- 整个标识符的长度不能超过64字符。

**原则2：所有命名都应遵循达意原则，即名称应含义清晰、明确、直观且可以拼读。**

说明：这些名字应该能反映它所代表的实际东西，具有一定的意义。

**原则3：命名规则中没有规定到的地方才可有个人命名风格，但使用时要自始至终保持一致，不可来回变化。**

说明：个人的命名风格，在符合所在项目组或产品组的命名规则的前提下，才可使用。（即命名规则中没有规定到的地方才可有个人命名风格）。

**原则4：命名必须在同一项目或产品中统一**

### 数据库命名规范

#### 命名涉及字符集合

a-z 0-9 和 \_ 共 37个，即所有的数据库对象均采用小写字母、阿拉伯数字、下划线组合的方式进行命名。

以字母开头。**功能点**建议不超过三个英文单词（模块前缀由 2-4 字符规则约束）；组合对象名（如外键 `fk_wo_workorder_cu_customer`）不受此限。

#### 数据库命名规则

前缀\_系统名称\[\_地区\]

##### 数据库前缀规则

数据库前缀仅用于**库名**，不用于表名。数仓库内的表仍按 `模块前缀_功能点` 命名，不再额外加 `ods_` 之类。

1. 业务数据库前缀： `app_`
2. 数仓数据前缀按层分前缀（均带下划线）
   * 贴源层: `ods_`
   * 公共维度层: `dim_`
   * 数据明细层: `dwd_`
   * 汇总数据层: `dws_`
   * 数据应用层: `ads_`
3. 外部数据库前缀: `ext_`

#### 数据库对象命名规则

规则：对象名字 = [前缀_]实际名称，各属性之间用下划线连接，长度不超过64字符。

名称应该清晰明了，能够准确表达事物的含义，最好可读，遵循"见名知意"的原则。

**Table 表命名规范：**

格式：`模块前缀_功能点`

- **大小写**：统一**全部小写**，与上文"所有数据库对象均采用小写字母、阿拉伯数字、下划线"的字符集合规则一致。模块前缀也写小写（`wo_workorder`，不是 `WO_WORKORDER`）。
- **分隔符**：模块前缀与功能点之间、功能点内多个单词之间，均使用下划线 `_` 分隔。
- **模块前缀长度**：控制在 **2-4 字符**，优先用通用缩写。
  - 推荐：`wo`、`cu`、`fin`、`sys`、`prod`、`inv`
  - 不推荐：`w`（过短，含义不清）、`customer`、`finance`（过长，挤压功能点空间，违背"前缀"用意）
- **不要与数据库对象前缀冲突**：模块前缀不得与下文「前缀命名规则」中的对象前缀同名，避免一眼看不出 `fn_invoice` 是函数还是表。已占用、需要避开的对象前缀：`t`、`v`、`mv`、`pkg`、`sp`、`trg`、`fn`、`pk`、`fk`、`uk`、`idx`、`job`、`seq`、`dbl`、`syn`、`tp`、`cst`、`tbs`、`p`。例如财务模块不要用 `fn`，建议改用 `fin`。
- **功能点词性**：使用名词或动宾短语，单数形式。
  - 单数：`wo_workorder`，不是 `wo_workorders`
  - 名词：`cu_customer`、`fin_invoice`
  - 动宾：`wo_assign_user`、`fin_refund_order`
- **功能点要自解释（重要）**：把模块前缀去掉后，剩下的功能点部分必须能让同事一眼看懂是什么业务数据。**实体类名是去前缀后的剩余部分**（Entity 类靠 `@TableName("模块前缀_功能点")` 绑定到具体表），太泛的功能点会污染整个 Java 代码层。
  - ✅ 自解释：`wo_workorder` → `Workorder`；`cu_follow_record` → `FollowRecord`；`fin_refund_order` → `RefundOrder`
  - ❌ 过泛：`wo_info` → `Info`、`cu_data` → `Data`、`fin_record` → `Record`、`wo_log` → `Log`（去前缀后完全不知道是什么）
  - ❌ 过抽象：`wo_item`、`cu_main`、`fin_detail`（item/main/detail 是结构词，不是业务词）
  - 经验法则：功能点本身就应该是一个**业务名词**，而不是"工单的什么东西"——读 `Workorder.java`、`Customer.java` 比读 `Info.java`、`Data.java` 要清楚 10 倍。
  - 同名功能点跨模块出现时（如 `wo_task` 和 `pm_task` 都生成 `Task`），靠 Java 包名隔离（`com.example.workorder.entity.Task` vs `com.example.pm.entity.Task`），不要为此在表名里加二级前缀。
- **通用表**：跨模块共享的基础表，以 `all_` 开头，如 `all_data_dict`、`all_sys_config`、`all_attachment`。
- **不要重复库前缀**：表名里不需要再带 `app_` / `ods_` 等库级前缀，那是数据库名的事。

**常用模块前缀参考**（各产品线可按需扩充，但同一项目内必须保持一致）：

| 前缀 | 模块 | 示例表名 |
|------|------|----------|
| `wo` | 工单 (Work Order) | `wo_workorder`、`wo_task`、`wo_attach_file`、`wo_assign_user` |
| `cu` | 客户 (Customer) | `cu_customer`、`cu_contact_person`、`cu_follow_record` |
| `fin` | 财务 (Finance) | `fin_invoice`、`fin_payment_record`、`fin_refund_order` |
| `sys` | 系统管理 | `sys_user`、`sys_role`、`sys_permission`、`sys_menu` |
| `org` | 组织架构 | `org_department`、`org_position` |
| `prod` | 产品 | `prod_product`、`prod_category`、`prod_sku` |
| `ord` | 订单 | `ord_order`、`ord_order_item` |
| `inv` | 库存 (Inventory) | `inv_stock`、`inv_stock_log` |
| `log` | 日志/审计 | `log_login`、`log_operation` |
| `all` | 通用/全局共享 | `all_data_dict`、`all_sys_config`、`all_attachment` |

> 项目落地时，建议把本项目实际使用的模块前缀列在项目自己的 README 或 wiki 里，避免新人凭感觉自创前缀（如同一个"客户"模块出现 `cu_` / `cust_` / `customer_` 混用）。

**各数据库标识符长度限制：**

| 数据库 | 表名 | 列名 | 索引名 / 约束名 |
|--------|------|------|----------------|
| MySQL | 64 | 64 | 64（超出直接报错：errno 1059 `Identifier name is too long`） |
| DB2 | 128 | 128 | 128 |
| Access | 64 | 64 | 64 |
| MSSQL | 128 | 128 | 128 |
| PgSQL | 63 字节 | 63 字节 | 63 字节（超出**静默截断**，不报错） |
| Oracle 12.2+ | 128 | 128 | 128（12.1 及以前均 30） |

- 所有标识符（库/表/列/索引/约束/序列）共用同一套上限，具体数字以目标库官方文档为准。
- **PgSQL 特别注意**：标识符在 63 字节处截断（`NAMEDATALEN-1`）且不报错，两个只差尾部的长索引名会撞成同一个对象，报 `relation already exists`。
- 表里是**报错线，不是预算**。索引名 = 前缀 + 表名 + 各字段名拼出来的，很容易撞线——取名前先算一遍，见下文[对象名长度与超长降级](#对象名长度与超长降级)。

**统一命名字段名：** → 已拆到 [field-conventions.md](field-conventions.md)（通用字段、扩展字段、`_at` / `_by` / `removed` / `version` / `sort_order` 约定）。本文件只管**对象与字段怎么取名**。

**字段命名规范：**

- 禁止数据库关键字**单独**作为字段名：`type`、`order`、`select`、`time`、`datetime`、`condition` 等。**组合词允许**：`order_no`、`order_status`、`order_time`、`condition_type` 都可以
- "组合词允许"只代表**不违反关键字规则**；若该字段属于统一字段范畴（创建/修改时间、逻辑删除、乐观锁、排序），仍必须用 [field-conventions.md](field-conventions.md) 里的固定名字，不要用允许的组合词自创 `create_time` / `update_time` / `is_deleted` 变体
- `name` 可用，但建议优先使用 `<语义>_name` 组合（如 `display_name`、`product_name`、`role_name`），避免单独的 `name` 在 SQL 里需要反引号 `` `name` `` 转义
- 字段名使用名词或动宾短语
- 字段名**单数**形式（`phone` 而非 `phones`，多值字段除外）
- 不重复表的名称——在 `cu_customer` 表里写 `phone`，不要写 `customer_phone`

##### 前缀命名规则

| **NO.** | **数据库对象**               | **前缀命名** | **示例** |
|---------|-------------------------|----------|---------|
| 1       | 表(table)                | 无（直接用模块前缀）    | `wo_workorder` |
| 2       | 视图(view)                | v        | `v_wo_overdue` |
| 3       | 物化视图(Materialized View) | mv       | `mv_daily_report` |
| 4       | 包(package)              | pkg      | `pkg_order_api` |
| 5       | 存储过程(procedure)         | sp       | `sp_sync_customer` |
| 6       | 触发器(trigger)            | trg      | `trg_wo_status_log` |
| 7       | 函数(function)            | fn       | `fn_calculate_age` |
| 8       | 主键(primary key)         | pk       | `pk_wo_workorder` |
| 9       | 外键(Foreign Key)         | fk       | `fk_wo_workorder_cu_customer` |
| 10      | 唯一键/约束(unique key)      | uk       | `uk_cu_customer_phone` |
| 11      | 索引(index)               | idx      | `idx_wo_workorder_status` |
| 12      | 作业(job)                 | job      | `job_cleanup_log` |
| 13      | 序列(Sequence)            | seq      | `seq_wo_workorder_id` |
| 14      | DBLINK                  | dbl      | `dbl_erp_prod` |
| 15      | 同义词(synch)              | syn      | `syn_erp_customer` |
| 16      | 自定义类型(type)             | tp       | `tp_address` |
| 17      | 约束(CONSTRAINT)          | cst      | `cst_wo_workorder_check_status` |
| 18      | 表空间(tablespace)         | tbs      | `tbs_app_data` |
| 19      | 分区（Partition）           | p        | `p_wo_workorder_2025` |

说明：

- 示例中的前缀与名称之间统一使用下划线 `_` 连接，保持全小写。
- **各对象的命名形式**（表名/视图带模块段，其余按下面的字段/语义段拼接）：
  - 表：无对象前缀，直接 `<模块前缀>_<功能点>` → `wo_workorder`
  - 视图：`v_<模块>_<语义>` → `v_wo_overdue`
  - 物化视图：`mv_<语义>` → `mv_daily_report`（跨模块报表可不带模块段）
  - 主键：`pk_<表名>` → `pk_wo_workorder`
  - 分区：`p_<表名>_<分区键>` → `p_wo_workorder_2025`
  - 序列：`seq_<表名>_<字段>` → `seq_wo_workorder_id`（多表共享时改用 `seq_<模块>_<业务含义>`）
- **索引/约束/外键的字段顺序约定**：
  - 索引 `idx_<表名>_<字段>`，复合索引按索引中字段顺序拼接：`idx_wo_workorder_status_priority`；拼出来超长怎么缩 → [对象名长度与超长降级](#对象名长度与超长降级)
  - 唯一键 `uk_<表名>_<字段>`，复合唯一键同上
  - 外键 `fk_<本表>_<引用表>`，方向是 from→to：`fk_wo_workorder_cu_customer` 表示 workorder 表引用 customer。**默认不建物理外键**，这个前缀仅供评审通过的例外场景使用，策略见 [foreign-keys.md](foreign-keys.md)
  - 约束 `cst_<表名>_<语义>`：`cst_wo_workorder_check_status`
- 序列名建议带上目标表名，如 `seq_wo_workorder_id`；若多个表共享序列，可用 `seq_<模块>_<业务含义>`。

##### 对象名长度与超长降级

适用于所有**由表名派生**的对象名：`idx_` / `uk_` / `fk_` / `cst_` / `seq_` / `pk_`。上限见上文[标识符长度表](#数据库对象命名规则)（多库部署取**最小值**）。

**先算预算，别等报错**：

```text
len(前缀_) + len(表名) + Σ(len(字段名) + 1)  ≤  目标库上限
```

- **表名建议 ≤ 30 字符**（硬上限仍是 64）。`fk_<本表>_<引用表>` 要把两个表名相加：`3 + 30 + 1 + 30 = 64`，表名再长一点就直接顶穿
- 复合索引最吃长度：每多一列就多 `<字段名>_` 一段，3-4 个长字段名很容易 70+

**原则：先缩词，后砍段；缩词先缩表名段。** 表名段（模块前缀 + 功能点）是对象名里最长的一段，**优先缩它的功能点单词**，字段名段能不动就不动；两者都缩了仍超，才砍段。缩写**单词**是无损的（模块、功能点、每个索引列都还能从名字读出来），砍掉整段才是丢信息。按下面的阶梯**逐级降级，降到不超就停**：

示例：表 `fin_customer_reconciliation`，索引列 `customer_id, reconcile_status, created_at`

| 级 | 做法 | 结果 | 长度 |
|----|------|------|------|
| 0 | 全拼 | `idx_fin_customer_reconciliation_customer_id_reconcile_status_created_at` | 71 ✗ |
| 1 | 先缩**表名段的功能点单词**（缩写规则见下），模块前缀 `fin` 原样保留（它是反查表的路由信息）。**表名段是最大的长度来源，先从它下手，字段名段一个字不动** | `idx_fin_cust_recon_customer_id_reconcile_status_created_at` | 58 ✓ MySQL/PgSQL 到此为止 |
| 2 | 仍超再缩**字段名段里的单词**，从**最后一列往前**缩；缩到 3-4 字母仍超就连前导列一起缩。**所有索引列仍出现在名字里** | `idx_fin_cust_recon_customer_id_recon_status_created` | 51 ✓ |
| 3 | 目标库更紧（Oracle 12.1 上限 30）才走：砍非前导列，留**前导列 + `_Ncol`**（N = 索引列数），完整列序写进 DDL 注释或索引 `COMMENT` | `idx_fin_cust_recon_cust_id_3col` | 31 ✗ |
| 4 | 兜底：`idx_<表名段>_<序号>`，序号 = 该表 DDL 中第几个索引（从 1 起） | `idx_fin_cust_recon_1` | 20 ✓ |

- 级 1-2 无损；只跑 MySQL / PgSQL 时**级 1 通常就够了**（表名一缩就掉进上限）。级 3-4 开始丢列信息，只给 Oracle 12.1（30）这类紧上限用
- `fk_<本表>_<引用表>` 有两段表名，级 1 对两段都适用：`fk_fin_customer_reconciliation_cu_customer`（42）→ `fk_fin_cust_recon_cu_cust`（25）
- **连级 4 都超 → 表名本身太长，回去改表名**（这正是"表名建议 ≤ 30"的原因）
- 降级后**仍须在本表内唯一**：撞名加 `_2` / `_3`（沿用级 4 的序号写法，不要另起一套）

**缩写怎么取**（原则同[基本命名原则](#基本命名原则)的原则1——完整单词优先，长了才缩；下面是索引名段的具体写法）

1. 公认缩写优先：`customer`→`cust`、`message`→`msg`、`amount`→`amt`、`quantity`→`qty`、`number`→`no`、`description`→`descr`、`configuration`→`config`、`reconciliation`→`recon`、`priority`→`prio`、`attachment`→`attach`
2. 没有公认缩写就取单词头 3-4 个字母：`settlement`→`sett`、`shipment`→`ship`
3. **同一个词全项目只用一种缩写**，和模块前缀表一起登记在项目 README / wiki，不要 `cust` / `cst` / `cmr` 混用
4. 不许用的缩写：撞 SQL 关键字（`desc` / `order` / `group` / `select`）、单字母（`_id` / `_no` 这类既定后缀除外）、拼音或拼音首字母（`kh` / `gd`）
5. **只缩派生名，不改对象本身**：表仍叫 `fin_customer_reconciliation`、字段仍叫 `customer_id`，统一字段（`created_at` / `removed` / `version`）更不许为凑长度改名——反查靠 DDL 里的 `ON <表名全名>` 和名字里的模块前缀

**任何一级都不许做的事：**

- **不许省掉表名段**（`idx_customer_id`、`idx_status` ✗）：索引名的**唯一范围各库不同**，见下表，按最严口径必须带表名；DBA 排查也靠名字反查表

  | 库 | 索引名唯一范围 | 不带表名段会怎样 |
  |---|---|---|
  | MySQL | 表内唯一 | 本库能建，换库就撞名 |
  | SQL Server | 表内唯一 | 同 MySQL |
  | PostgreSQL | **schema 内唯一** | 第二张表上建同名索引直接报 already exists |
  | Oracle | **schema 内唯一** | 同 PostgreSQL |
  | 达梦 DM | 按最严口径视为 schema 内唯一 | 信创上线前逐库验证，不要照搬 MySQL 结论 |
  | H2 | 表内唯一 | 测试库不报错，**掩盖 PG / Oracle 上的问题** |

  → **统一都带表名段**：既是跨库可移植的必要条件，也是 DBA 从索引名反查表的唯一线索。**别拿 H2 的宽松行为当准**——它是表内唯一，写成 `idx_status` 在测试里一路绿灯，到 PG / Oracle 上才炸
- **不许加随机 / 哈希后缀**（`idx_wo_workorder_a1b2` ✗）：不可读，且脚本重跑一次名字就变，diff 和迁移都不可复现
- **不许靠数据库自动命名**（MySQL `ADD INDEX (a, b)` 默认取名 `a`，重名自动加 `_2`）：跨库行为不一致，评审时也看不出索引用途
- **上线后不要改索引名**：改名 = `DROP INDEX` + `CREATE INDEX`，大表上代价很高——初版就按阶梯留足余量
