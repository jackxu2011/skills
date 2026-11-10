# SQL 编写规范详解

需要缩进风格、空格换行、CREATE TABLE 排版、`SELECT *` 例外的完整示例时加载。

**适用范围**：会话中编写或评审的一切 SQL 文本——MyBatis XML、注解 SQL（`@Select` 等）、Flyway 迁移脚本、临时查询 SQL，不限于建表 DDL。

**不适用**：`QueryWrapper` 等编程式构造由框架生成 SQL，本节管不到；统一字段的取名见 [field-conventions.md](field-conventions.md)，不在本文范围。

## SQL编写规范

**规则：所有关键字必须大写。** 如 `INSERT`、`SELECT`、`FROM`、`WHERE`、`GROUP BY`、`ORDER BY`、`CREATE`、`TABLE` 等。

**规则：所有函数及其参数中除用户变量以外的部分必须大写。**

### 缩进风格

**规则：脚本代码块采用缩进风格书写，保证代码清晰易读，风格一致。缩进格数统一为2个空格，但需要关键字右对齐或者参数/字段向上对齐的时候例外。**

示例：
```sql
CREATE SEQUENCE seq_wo_workorder_id
  MINVALUE 1
  START WITH 1
  INCREMENT BY 1;
```

**规则：同一条语句占用多于一行时，每行的第一个关键字应当右对齐，但INSERT...SELECT语句例外。**

示例：
```sql
SELECT field1, field2, ...
  FROM wo_workorder
 WHERE field1 > 1
   AND (field2 < CURRENT_DATE OR field2 > CURRENT_DATE + 3)
   AND field3 LIKE 'example%'
 GROUP BY field1, field2
 ORDER BY field1, field2;
```

**规则：在INSERT...SELECT语句中，如果需要换行时，应使INSERT INTO部分的字段与SELECT部分的字段一一对应，以增强可读性。**

示例：

错误的写法：
```sql
  INSERT INTO wo_workorder (dest_field1, dest_field2,
  dest_field3, dest_field4, dest_field5,
  dest_field6, dest_field7)
  SELECT src_field1, src_field2, src_field3,
  src_field4, src_field5, src_field6,
  src_field7
  FROM cu_customer;
```
正确的写法A：

```sql
  INSERT INTO wo_workorder (dest_field1, dest_field2,
         dest_field3, dest_field4, dest_field5,
         dest_field6, dest_field7)
  SELECT src_field1, src_field2,
         src_field3, src_field4, src_field5,
         src_field6, src_field7
  FROM cu_customer;
```
正确的写法B：

```sql
  INSERT INTO wo_workorder (dest_field1, -- 注释
         dest_field2, -- 注释
         dest_field3, -- 注释
         dest_field4, -- 注释
         dest_field5, -- 注释
         dest_field6, -- 注释
         dest_field7) -- 注释
  SELECT src_field1, -- 注释
         src_field2, -- 注释
         src_field3, -- 注释
         src_field4, -- 注释
         src_field5, -- 注释
         src_field6, -- 注释
         src_field7 -- 注释
  FROM cu_customer;

```

说明：

- 在错误的写法中，虽然SELECT语句部分的src_field1, src_field2, src_field3可以写在一行中，但由于INSERT INTO语句部分中dest_field1, dest_field2写在一行中，而dest_field3在下一行，因此SELECT语句中每行的字段应与INSERT INTO语句中的字段一一对应（如正确的写法A）；
- INSERT INTO语句中的各个字段折行后，应缩进并与INTO关键字左对齐（如正确的写法B）；
- SELECT语句中折行后的第一个字段名应缩进并与上一行的第一个字段名左对齐。

**规则：INSERT INTO语句中，如果需要对每个字段增加注释，应将每个字段单独列为一行，并在行尾增加注释。**

示例1：

```sql
INSERT INTO wo_workorder (dest_field1, -- 注释1
       dest_field2, -- 注释2
       dest_field3) -- 注释3
VALUES (field_value1,
        field_value2,
        field_value3);
```
或者
```sql
INSERT INTO wo_workorder (
       dest_field1, -- 注释1
       dest_field2, -- 注释2
       dest_field3) -- 注释3
VALUES (
  field_value1,
  field_value2,
  field_value3);
```
或者
```sql
INSERT INTO wo_workorder
(
  dest_field1, -- 注释1
  dest_field2, -- 注释2
  dest_field3 -- 注释3
)
VALUES
(
  field_value1,
  field_value2,
  field_value3
);
```
**其他说明：**

- 在规范的代码示例中详细给出了各种情况下的范例写法，请编码过程参考；
- 在数据库脚本的编码过程中，请严格按照代码示例进行书写；
- 对于模板中未涉及的情况，应在保证符合上述规范原则，同时和代码示例中的风格保持一致的前提下，根据实际情况灵活处理。

### 空格及换行

**建议：**建议对超过120字符的语句要分行书写，长表达式应在低优先级操作符处换行，操作符或关键字放在新行之首。划分出的新行应适当地缩进，使排版整齐、语句可读。是否分行应根据实际情况而定，原则是保证代码整齐、语句可读。

分行示例：
```text
                    120字符
                       ↓
…… (A * B * C * D) + (E * F) + ……
```

应写成：
```text
…… (A * B * C * D)
+ (E * F) + ……
```
说明：

- 加法的优先级低于乘法，因此应在加号处换行；
- 两组乘法虽然在逻辑上会先于加法执行，但显式加上括号使可读性更强。

**规则：双目运算符、操作符前后应以空格分隔，间隔符之后应以空格分隔。**

**建议：**不同优先级的操作符混合使用时，建议使用括号进行隔离。

示例：
```text
…… A * B + C ……
```


应写成：
```text
…… (A * B) + C ……
```

说明：使用括号使代码的优先级更加清晰，而且可以避免犯错。

**建议：**SQL语句中存在不同优先级的关系运算符时，建议使用括号。

示例：
```sql
WHERE field1 > 1 AND field2 < CURRENT_DATE OR field3 > CURRENT_DATE + 3
```

应写成：
```sql
WHERE (field1 > 1 AND field2 < CURRENT_DATE) OR (field3 > CURRENT_DATE + 3)
```

说明：如果SQL语句的条件复杂，很容易出现问题，因此增加括号可以提高可阅读性。

### 创建表

**规则：脚本中的字段名缩进为2个空格。**

**规则：脚本中字段名称、字段类型、DEFAULT关键字间向左靠。不要对齐，以方便git等版本管理软件合并**

示例：
```sql
CREATE TABLE wo_workorder
(
  int_field INT NOT NULL,
  float_field DECIMAL(4, 2) DEFAULT 1.23 NOT NULL,
  date_field DATE DEFAULT NULL,
  varchar_field VARCHAR(200) NULL
);
```

### 符号\*

**规则：脚本中不允许出现"\*"的用法，必须用实际的字段名代替，INSERT语句必须指定要插入的字段名。**

示例：

1、SELECT 语句

错误用法：
```sql
SELECT * FROM wo_workorder WHERE ...
```

正确用法：
```sql
SELECT field1, field2, ... FROM wo_workorder WHERE ...
```

2、INSERT 语句

错误用法：
```sql
INSERT INTO wo_workorder VALUES ...
```

正确用法：
```sql
INSERT INTO wo_workorder (field1, field2, ...) VALUES ...
```

## 统一使用UTF-8字符集

规则：所有数据库统一使用 UTF-8 字符集，MySQL 使用 `utf8mb4` 编码以支持 emoji 等扩展字符。
