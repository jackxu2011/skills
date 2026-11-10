---
name: git-commit
description: 创建符合 Conventional Commits 规范的 git commit（可选 push），自动分析 git diff 生成 message。参数给的是「要提交什么」的意图，由 AI 组织成规范 message，不原样当 message 用。意图识别优先级：$ARGUMENTS > staged 文件 > 同意图的 unstaged/untracked。大模块/长任务收尾调用前，先自己提炼一句提交意图再调用本技能。不追加 AI Co-Authored-By。Use when 用户表达提交/commit 意图（含对"是否提交"的任何肯定答复）；同一请求含 push/推 等推送词时 commit 后追加 git push。单独 push 不触发本技能。
allowed-tools: Bash(git status *) Bash(git add *) Bash(git commit *) Bash(git diff *) Bash(git push *) Bash(git rev-parse *) Bash(git -C * status *) Bash(git -C * add *) Bash(git -C * commit *) Bash(git -C * diff *) Bash(git -C * push *) Bash(git -C * rev-parse *) AskUserQuestion
model: sonnet
argument-hint: "<optional: 本次提交的内容/意图>"
metadata:
  version: "1.0.0"
---

Create a git commit.

## AI Attribution 由工具配置决定

加不加 AI 署名 **由工具配置决定，不由你决定**。你既不加、也不删：

- 不要写 `Co-Authored-By: Claude ...`、`Generated with Claude Code`、`sessionUrl` 链接，或任何形式的 AI 联合作者 / 来源
  trailer
- 不要改写或删除 message 里已有的 trailer——那可能是模板或协作者写的真署名
- **例外**：仅当用户 **显式要求**（如明确说出"加上 Claude Co-Authored-By"）时才追加
- **适用范围**：所有 `git commit` 操作

## worktree 模式：所有 git 命令必须带 `-C`

在 worktree 会话中，裸 `git` 可能解析到主仓库而非当前 worktree，提交 / 推送会落到错误的分支。

- **判定**：`git rev-parse --git-dir` 与 `git rev-parse --git-common-dir` 输出不同 → 当前在 linked worktree（主仓库下两者相同）
- **行为**：一旦判定在 worktree，本技能中 **每一条** git 命令都写成 `git -C <worktree 根目录> <subcommand>`——`status` /
  `add` / `diff` / `commit` / `push` / `rev-parse` 无一例外
- **根目录取值**：环境块里的工作目录即 worktree 路径；拿不准时用 `git -C <该路径> rev-parse --show-toplevel`
  取绝对根目录，后续命令统一复用它
- **不要**用 `cd <worktree> && git ...` 代替 `-C`——Bash 工具的 cwd 在多次调用间会漂移，`-C` 是唯一显式且稳定的写法

## Steps

0. **确定 git 前缀**：按上一节判定是否在 worktree——是则下面每条命令都用 `git -C <worktree 根目录>`，否则用裸 `git`

1. **识别提交意图**（按优先级，先命中先用，不要跳级）：
  1. **`$ARGUMENTS`（最高优先级）**：它是"这次要提交什么、为什么"的 **范围说明**，不是成品 message，贯穿步骤 3、5、6
  2. **staged 文件**（`git diff --staged`）：没有参数时，从已 staged 的改动反推意图——用户特意 stage 的东西就是他想提交的主题
  3. **unstaged + untracked**：只用来补全和佐证上面的意图， **不单独定调**；只有在既没参数又没 staged 时，才从它们反推意图，且必须先跟用户确认

  - 三者都给不出一个明确意图 → 用 `AskUserQuestion` 问，不要猜
  - **大模块 / 长任务收尾调用本技能时**：先自己提炼一句提交意图（这次要提交什么、为什么）再调用，作为 `$ARGUMENTS`
    传入——不要把一堆改动丢进来让本技能从 diff 里猜主题

2. **完整看 git 状态**：`git status --short`，列出所有 staged + unstaged + untracked 文件， **不要跳过这一步**

3. **按意图收文件**（优先级固定：staged → 同意图的 unstaged/untracked → 跨主题拆 commit）：
  - **意图只覆盖部分改动时，只取那部分**，其余留在工作区——意图是取舍的边界，不是"全都提"
  - **第一步取 staged 中落在意图内的文件**
  - **第二步扫 unstaged + untracked，把与该意图明确同主题的纳入本次提交**（如实现已 staged、配套测试/文档还没 add），一起提交
  - **跨主题**（改动同时命中多个不相干的意图，或工作区里有与本次意图无关的改动）： **首选拆成多次提交**——用
    `AskUserQuestion` 把待选文件按主题分组列给用户确认，然后 **每组各走一遍步骤 3–7**，一次 commit
    只落一个主题；只有用户明确说"一起提"才合并，绝不擅自混进一次 commit
  - **完全空**（没 staged 也没别的改动）：报告 "nothing to commit" 并停止
  - **判定不准时优先问**，不要替用户决定

4. 仅 `git add` 你 **确认要提交**的文件—— **严禁 `git add .` 或 `git add -A`**

5. **Review 范围**：`git diff --staged --stat`，必要时看 `git diff --staged` 的实际内容，确认文件范围与步骤 1 的意图一致。
   **`--stat` 只给行数、看不出改了什么，不能只看它就往下走**。范围对不上就回到步骤 3 重新取舍，不要带着错的范围往下走

6. **组织 commit message**：意图和 diff **都是输入，都不是成品**
  - `$ARGUMENTS` 给语义与范围，`git diff --staged`（无 staged 时用 `git diff`）给事实
  - 按上面的 Commit Message Format 重新组织—— **严禁把 `$ARGUMENTS` 原样塞进 `git commit -m`**
  - 两者对不上时以 diff 为准，并向用户说明你改了什么
  - **写完回头核一遍**：把 message 与 `git diff --staged` 的实际内容逐条对上（type / scope / 描述主体 / 是否
    breaking）。对不上就 **改 message**；只有当发现是文件取舍错了，才回步骤 3 改范围

7. **Create commit**：用步骤 6 生成的 message 提交。拆分提交时，每组改动各走一遍步骤 3–7，一次 commit 一个主题

8. 若同一请求含 "push"、"推"、"推送"、"and push" 等推送意图，执行 `git push`；按 ## Output 报告

## 主题判定指引

判定"是否同主题"看两点——拿不准就用 AskUserQuestion：

| 同主题（可加入本次 commit） | 跨主题（必须拆 commit / 问用户） |
|----------------------|----------------------------------|
| 同一 feature 的实现 + 配套测试 + 文档 | 一次 feature 改动 + 顺手改的 typo / 注释 / 配置 |
| 同一文件的多处修改都服务于本次目标 | 同一文件里**不同段落**做不同事情（如改 bug + 重构函数签名） |
| 同一个 Starter / skill 的相关文件 | 不同模块 / 不同 skill 的改动 |
| 用户明说"加上 X 一起提交" | 用户没指明范围时，工作区里有任意 unrelated 改动 |

> 默认行为： **staged 优先**；unstaged/untracked 中与本次意图同主题的 **主动纳入**，不同主题的保持不动，留给用户自己决定。

> 经验教训： **stage + unstaged 混杂是一次 commit 的高发污染场景**——AI 倾向于把所有"看起来相关"的都加进来，结果一次 commit
> 覆盖多个语义不同的目标，破坏 git bisect / revert 粒度。宁可多问一句，不要混。

## Commit Message Format

- Structure: `<type>[optional scope]: <description>`
- Start with a required type:
  - `feat:` – new feature (triggers `MINOR` version bump)
  - `fix:` – bug fix (triggers `PATCH` version bump)
  - Others allowed: `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `chore:`, `build:`, `ci:`
- Optional `!` after type/scope, or `BREAKING CHANGE:` footer indicates a breaking change (triggers `MAJOR` version
  bump)
- Optional scope in parentheses: e.g., `feat(auth):` or `fix(parser):`
- Description: use imperative, present tense; lowercase start; no trailing period
- First line ≤ 72 characters; blank line before optional body/footer
- Example: `feat: add user authentication with JWT`
- Full specification: see [references/commit.md](references/commit.md)

## Output

Show a brief confirmation:
✓ Committed: [commit message] [number] files changed
✓ Pushed: [branch] → [remote]（仅执行了 push 时输出）
