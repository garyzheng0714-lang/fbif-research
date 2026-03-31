---
name: batch-research
description: "一键启动批量品牌调研。从飞书多维表格读取待调研品牌清单，逐个启动独立 Claude Code 会话执行 fbif-research 调研，调研完成后自动回写结果到飞书。触发词：批量调研、batch research、自动调研、开始调研、调研所有品牌、按列表调研、一键调研。"
---

# 批量品牌调研

从飞书多维表格读取待调研品牌，逐个自动执行 fbif-research 调研，完成后回写结果。

## 依赖

- `~/.claude/skills/fbif-research/` — 调研 skill（含脚本和配置）
- `~/.claude/skills/fbif-research/bitable-config.json` — 飞书凭证
- `~/.claude/skills/fbif-research/research-loop.sh` — 主循环脚本

## 执行流程

### 第1步：检查环境

```bash
echo "=== 环境检查 ==="
SKILL_DIR="$HOME/.claude/skills/fbif-research"
errors=0

# 检查核心文件
for f in "$SKILL_DIR/research-loop.sh" "$SKILL_DIR/bitable-config.json" "$SKILL_DIR/scripts/bitable_read.py" "$SKILL_DIR/scripts/bitable_write.py" "$SKILL_DIR/scripts/validate_completion.py"; do
  if [ ! -f "$f" ]; then
    echo "MISSING: $f"
    errors=$((errors+1))
  fi
done

# 检查 claude CLI
which claude >/dev/null 2>&1 || { echo "MISSING: claude CLI"; errors=$((errors+1)); }

# 检查 Python 依赖
python3 -c "import oss2" 2>/dev/null || echo "WARNING: oss2 未安装 (pip install oss2)"

[ $errors -eq 0 ] && echo "✓ 环境检查通过" || echo "✗ 缺少 $errors 个文件"
```

如果环境检查不通过，提示用户先安装 fbif-research skill。

### 第2步：查看待调研清单

```bash
python3 ~/.claude/skills/fbif-research/scripts/bitable_read.py
```

展示待调研品牌列表，确认品牌数量和顺序。

### 第3步：检查残留状态

```bash
echo "=== 锁文件检查 ==="
LOCK_DIR="$HOME/.claude/skills/fbif-research/.locks"
if ls "$LOCK_DIR"/*.lock 2>/dev/null; then
  echo "发现残留锁文件（上次中断？）"
  echo "选项: 删除锁文件重新调研 / 跳过已锁品牌继续"
else
  echo "✓ 无残留锁文件"
fi
```

如果有残留锁文件，用 AskUserQuestion 问用户：
- A) 清除所有锁文件，从头开始
- B) 保留锁文件，跳过已锁品牌继续
- C) 取消

选 A 则执行 `rm -f ~/.claude/skills/fbif-research/.locks/*.lock`

### 第4步：启动批量调研

用 AskUserQuestion 确认启动：

> 即将启动批量调研，共 {N} 个品牌待调研。
> 每个品牌约 30-90 分钟，全部完成预计 {N*1} - {N*1.5} 小时。
> 调研将在后台运行，日志保存在 ~/.claude/skills/fbif-research/logs/
>
> A) 启动（推荐在 tmux/screen 里运行）
> B) 先用 dry-run 预览
> C) 取消

选 A：

```bash
nohup bash ~/.claude/skills/fbif-research/research-loop.sh > ~/.claude/skills/fbif-research/logs/batch-$(date +%Y%m%d-%H%M%S).log 2>&1 &
BATCH_PID=$!
echo "✓ 批量调研已启动 (PID: $BATCH_PID)"
echo "  日志: tail -f ~/.claude/skills/fbif-research/logs/batch-*.log"
echo "  停止: kill $BATCH_PID"
echo "$BATCH_PID" > ~/.claude/skills/fbif-research/.batch-pid
```

选 B：

```bash
bash ~/.claude/skills/fbif-research/research-loop.sh --dry-run
```

### 第5步：输出监控命令

启动后输出以下信息：

```
批量调研已在后台启动。

监控命令：
  查看进度: tail -f ~/.claude/skills/fbif-research/logs/batch-*.log
  查看锁文件: ls ~/.claude/skills/fbif-research/.locks/
  查看待调研: python3 ~/.claude/skills/fbif-research/scripts/bitable_read.py
  停止调研: kill $(cat ~/.claude/skills/fbif-research/.batch-pid)
  清除锁文件: rm ~/.claude/skills/fbif-research/.locks/*.lock

飞书多维表格会实时更新调研状态。
```

## 子命令

- `/batch-research` — 启动批量调研（默认）
- `/batch-research status` — 查看当前进度
- `/batch-research stop` — 停止正在运行的批量调研
- `/batch-research clean` — 清除所有锁文件

### status 子命令

```bash
echo "=== 批量调研状态 ==="
# 检查进程
if [ -f ~/.claude/skills/fbif-research/.batch-pid ]; then
  PID=$(cat ~/.claude/skills/fbif-research/.batch-pid)
  if kill -0 "$PID" 2>/dev/null; then
    echo "✓ 运行中 (PID: $PID)"
  else
    echo "✗ 进程已结束"
  fi
else
  echo "✗ 未启动"
fi

# 锁文件
echo ""
echo "=== 品牌状态 ==="
ls ~/.claude/skills/fbif-research/.locks/*.lock 2>/dev/null && echo "(以上品牌正在调研或已失败)" || echo "无锁文件"

# 待调研
echo ""
python3 ~/.claude/skills/fbif-research/scripts/bitable_read.py
```

### stop 子命令

```bash
if [ -f ~/.claude/skills/fbif-research/.batch-pid ]; then
  PID=$(cat ~/.claude/skills/fbif-research/.batch-pid)
  kill "$PID" 2>/dev/null && echo "✓ 已停止 (PID: $PID)" || echo "进程已结束"
  rm -f ~/.claude/skills/fbif-research/.batch-pid
else
  echo "未找到运行中的批量调研"
fi
```

### clean 子命令

```bash
rm -f ~/.claude/skills/fbif-research/.locks/*.lock 2>/dev/null
echo "✓ 已清除所有锁文件"
```
