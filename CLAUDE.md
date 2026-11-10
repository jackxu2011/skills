# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Claude Code 相关

### skill frontmatter 里的 Claude Code 专属字段

除通用字段（`name` / `description` / `metadata.version`）外，SKILL.md 还支持 Claude Code 专属字段，写 skill 时按 Claude Code 的 skill 规范填：

- `allowed-tools` —— 该 skill 可用的工具白名单，Bash 用通配限定（如 `Bash(git status *)`）。**skill 里要跑什么命令就在这里列全**，否则执行时会反复弹权限确认
- `model` —— 覆盖默认模型（如 `model: sonnet`），只给对成本/速度敏感的 skill 用
- `argument-hint` —— 斜杠命令的参数提示，仅在 skill 支持 `$ARGUMENTS` 时写

### 提交走 git-commit skill

本仓库的提交直接走 `skills/git-commit` 的流程（Conventional Commits，可选 push），不要手写 commit message 流程。同一请求含"push / 推"等推送词时，commit 后追加 `git push`；单独 push 不触发。

### 渐进式披露对上下文的影响

Claude Code 按需加载 `references/` 下的文件，SKILL.md 本身常驻上下文。因此 SKILL.md 里**不要内联细则**——多写的每一行都会在每次触发时消耗上下文；索引表指向具体文件 + 具体锚点即可。
