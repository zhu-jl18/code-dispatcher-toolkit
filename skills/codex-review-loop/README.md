# Codex Review Loop 模板

可复制粘贴的 Claude Code review-loop 模板（Codex 版）：
- `/codex-review-loop <任务>` 让 Claude 先实现任务
- Claude `stop` 时，Stop hook 调用 Codex 做独立多视角 review（Diff + Holistic + 条件分支）
- 生成 review 后 Claude 继续处理，再次 stop 才退出

## 模板内容

- `settings.json`：注册 Stop hook
- `hooks/codex-review-loop-stop.sh`：Stop hook 脚本
- `commands/codex-review-loop.md`：启动 loop
- `commands/cancel-codex-review-loop.md`：取消 loop（主命令）

## 安装

将本目录下的 `commands/`、`hooks/` 复制到你项目的 `.claude/` 目录：

```bash
cp -r commands/ <your-project>/.claude/commands/
cp -r hooks/    <your-project>/.claude/hooks/
chmod +x <your-project>/.claude/hooks/codex-review-loop-stop.sh
```

然后把 `settings.json` 里的 `hooks.Stop` 合并到项目的 `.claude/settings.json` 或 `.claude/settings.local.json`。

安装后结构：

```text
<your-project>/.claude/
├── commands/
│   ├── codex-review-loop.md
│   └── cancel-codex-review-loop.md
├── hooks/
│   └── codex-review-loop-stop.sh
└── settings.json (or settings.local.json)
```

## 依赖

- `codex` CLI 在 PATH 中（`npm install -g @openai/codex`）
- `~/.codex/config.toml` 需启用：

```toml
[features]
multi_agent = true
```

不依赖 `code-dispatcher`。

## 使用

启动 loop：

```text
/codex-review-loop Implement feature X with tests
```

取消：

```text
/cancel-codex-review-loop
```

## 产物

- 状态文件：`.claude/review-loop.local.md`
- review 文件：`reviews/review-<id>.md`
- 日志：`.claude/review-loop.log`

建议将 `reviews/` 加入项目 `.gitignore`，避免 review 产物混入提交。
