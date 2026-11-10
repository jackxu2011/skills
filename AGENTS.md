# AGENTS.md

本仓库是 Agent Skills 开发仓库：产出的是给编码智能体用的 skill（Markdown 指令包），不是可运行代码。

## 仓库性质

没有 build / lint / test 工具链——正确性靠 SKILL.md 自带的「产出前自检」清单 + 人工评审保证，**不要发明构建命令**。IDE 配置（`.idea/`、`.vscode/`）已被 gitignore。

提交遵循 Conventional Commits，**不加 AI 署名**（不写 `Co-Authored-By`、`Generated with ...`、session 链接等任何形式的联合作者 / 来源 trailer），也不要改动 message 里已有的 trailer。

## 目录结构

```
skills/<skill-name>/SKILL.md          # skill 入口，必有
skills/<skill-name>/references/*.md   # 按主题拆的细则，可选
```

skill 名 = 目录名，全小写连字符。当前 skill：`database-design-standards`、`ui-feature-spec`、`git-commit`。

## skill 的编写约定（跨文件才能看出来的部分）

### 以 Agent Skills 规范为基线

写 / 改 skill 必须遵循 **[Agent Skills 规范](https://agentskills.io/specification)**：目录布局、SKILL.md 的 frontmatter 字段、`name` / `description` 的格式与长度约束、渐进式披露的写法，都以该规范为准。下面这些是本仓库在规范之上的**补充约定**，两者冲突时以规范为准。

### 两级结构：SKILL.md 是索引，references/ 是细则

每个 SKILL.md 只放四样东西：**硬约束 + Quick Start（取名/操作的 N 步）+ 产出前自检清单 + 索引表**。细则按主题一文件拆进 `references/`，SKILL.md 里写明「只读当前任务需要的那一个文件」——这是渐进式披露（progressive disclosure），目的是控制上下文占用，**不要把细则回填进 SKILL.md**。

自检清单是硬约束的可勾选版本，改硬约束时两边要同步。

### frontmatter 是 skill 的触发契约

`description` 承担全部触发职责，必须同时含三段：

1. **是什么**（一句概括能力边界）
2. **Use when …**（触发场景枚举，中英混合都行）
3. **负面边界**（"不用于…" / "不做…"，防止误触发）

**`metadata.version` 现阶段统一保持 `1.0.0`，不要改**——等第一次正式推送（push）之后，才开始按 SemVer bump（`fix` → patch、`feat` → minor、破坏性变更 → major）。在此之前改动 skill 内容也保持 `1.0.0`。

### 与其它 skill 的边界表

每个 SKILL.md 末尾有「与其它 skill 的边界」两列表，写明本 skill 管什么、哪些相邻主题交给别的 skill。**新增 skill 或扩写现有 skill 的职责时必须更新这张表**，避免职责重叠。

引用规则：指向**本仓库内**的 skill 用 skill 名（如 `database-design-standards`）；指向**本仓库外**的 skill 用主题表述（如「API 契约类 skill」「Java 编码规范类 skill」），不要硬编码外部 skill 名——外部 skill 不随本仓库分发，写死名会让引用悬空。

### 内容语言与风格

正文中文，硬约束条目用祈使句 + 具体反例（`i_status` ✗ / `wo_workorder` ✓）。规则要写到**可判定**——含长度上限、目标库差异（MySQL 64 vs PgSQL 63 静默截断）、降级阶梯，不能写"尽量""建议保持一致"这种无法自检的话。

## 修改 skill 时的校验点

- 符合 [Agent Skills 规范](https://agentskills.io/specification)（frontmatter 字段合法、`name` / `description` 格式与长度合规）
- frontmatter 的 `name` 与目录名一致
- SKILL.md 里每个 `references/*.md` 链接真实存在，锚点（`#章节名`）能对上
- 硬约束 / 自检清单 / references 三处说法一致
- 边界表覆盖了本次改动涉及的相邻 skill
- `metadata.version` 仍是 `1.0.0`（第一次正式推送前不 bump）
