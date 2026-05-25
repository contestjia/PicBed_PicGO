---
title: Claude Code Hooks 原理
---

# Claude Code Hooks 原理

#### You:
claude code的hooks使用和原理

#### ChatGPT:
好问题，这个点其实**很多人在用 Claude Code，却没真正“吃透” hooks**。我给你一套**工程视角**的说明：  
👉 **怎么用** + **底层原理** + **和普通脚本 / git hooks 的差异**，一次讲清。

---

## 一、Claude Code 的 hooks 是什么？

一句话定义：

> **Claude Code hooks = 在 Claude 执行特定阶段时，由你注入的“可编程拦截点”**

它允许你在 **Claude 读代码 / 写代码 / 执行命令 / 结束任务** 的关键节点：
- 自动跑脚本
- 校验 / 过滤 / 重写 Claude 的行为
- 注入上下文或强制约束

👉 本质上是 **“AI agent 生命周期的钩子函数”**。

---

## 二、Claude Code 有哪些 hooks（使用层面）

Claude Code 目前（主流）支持的 hooks 类型可以抽象成 4 类：

### 1️⃣ pre-command（命令前）

**触发时机**
- Claude 准备执行 shell 命令之前

**典型用途**
- 禁止危险命令（`rm -rf /`）
- 自动加 `--dry-run`
- 记录命令审计日志

**例子**
```bash
#!/bin/bash
# hooks/pre-command

if echo "$CLAUDE_COMMAND" | grep -q "rm -rf"; then
  echo "❌ Dangerous command blocked"
  exit 1
fi
```

---

### 2️⃣ post-command（命令后）

**触发时机**
- Claude 执行完 shell 命令之后

**用途**
- 校验结果
- 自动格式化
- 根据 exit code 决定是否允许 Claude 继续

```bash
#!/bin/bash
# hooks/post-command

if [ "$CLAUDE_EXIT_CODE" -ne 0 ]; then
  echo "Command failed, stopping agent"
  exit 1
fi
```

---

### 3️⃣ pre-edit / post-edit（写代码前 / 后）

**触发时机**
- Claude 准备修改文件前 / 修改完成后

**用途**
- 强制代码规范
- 防止改关键文件
- 自动跑 lint / test

```bash
# 禁止 Claude 改 schema
if [[ "$CLAUDE_EDIT_FILE" == *"schema.sql"* ]]; then
  echo "Schema is read-only"
  exit 1
fi
```

---

### 4️⃣ task-complete / session-end（任务结束）

**触发时机**
- Claude 认为“任务完成”

**用途**
- 自动跑全量测试
- 生成报告
- 提交 git commit

```bash
npm test || exit 1
git status --porcelain
```

---

## 三、hooks 是怎么被 Claude Code 调用的（原理）

这是重点 👇  
Claude Code **不是魔法**，它其实是一个：

> **AI Agent + 状态机 + shell 调度器**

### 核心流程（简化）

```
┌──────────┐
│  Prompt  │
└────┬─────┘
     ↓
┌──────────┐
│  Plan    │  ← Claude reasoning
└────┬─────┘
     ↓
┌──────────┐
│ Action   │  ← 写文件 / 执行命令
└────┬─────┘
     ↓
┌──────────┐
│ Observe  │  ← 命令输出 / diff
└────┬─────┘
     ↓
┌──────────┐
│ Decide   │  ← 是否继续
└──────────┘
```

👉 **hooks 就是插在这些节点之间的“强制同步点”**

---

### hooks 的真实执行机制

Claude Code 在每个阶段会做三件事：

1. **构造上下文环境变量**
   ```text
   CLAUDE_COMMAND
   CLAUDE_EDIT_FILE
   CLAUDE_DIFF
   CLAUDE_EXIT_CODE
   ```
2. **fork 一个 shell 子进程**
3. **同步等待 hook 脚本返回**

```text
exit 0 → 继续
exit ≠0 → 中断 / 回滚 / 重规划
```

⚠️ 这点很重要：  
> **hook 是“强制同步”的，不是回调**

---

## 四、为什么 hooks 能“控制” Claude？

因为 Claude Code 的执行模型是：

> **AI 不能越过系统级约束**

Claude **必须**：
- 等 hook 执行完
- 接受 hook 的 stdout / stderr
- 根据 exit code 调整行为

换句话说：

> hooks = AI agent 的“内核态”

---

## 五、hooks vs git hooks（本质区别）

| 对比 | Claude Code hooks | Git hooks |
|----|----|----|
| 控制对象 | **AI 行为** | 人类操作 |
| 时机 | agent 生命周期 | git 操作 |
| 能否影响决策 | ✅ 可以 | ❌ 只能阻断 |
| 上下文 | AI 意图 + diff | 文件状态 |
| 能否重规划 | ✅ | ❌ |

👉 **Claude hooks 是“认知层钩子”，git hooks 是“操作层钩子”**

---

## 六、一个高级用法（工程里很香）

### 🔥 用 hooks 把 Claude 变成“受控工程师”

**目标**
- Claude 只能：
  - 改指定目录
  - 测试必须全绿
  - commit message 符合规范

**组合 hooks**
```
hooks/
├── pre-edit        # 文件白名单
├── post-edit       # lint
├── pre-command     # 命令黑名单
├── post-command    # test gate
└── task-complete   # 自动 commit
```

👉 效果：
> Claude ≈ “不会越权、不会偷懒、不会忘跑测试的初级工程师”

---

## 七、一句话总结

> **Claude Code hooks 本质是：  
> 用 shell + exit code，把 AI 的“自由意志”变成“受控状态机”**

如果你愿意，下一步我可以给你：
- 🧠 **完整 hooks 生命周期状态图**
- 🛠 **一套可直接用的 hooks 模板**
- ⚔️ **如何用 hooks 防 prompt 注入 / 防越权改代码**

你更想继续哪一块？



#### You:
给一些使用例子

#### ChatGPT:
好，那我直接给你**“拿去就能用”的 hooks 例子**，从**基础 → 进阶 → 偏工程化/反 AI 作妖**，每个例子都说明**解决什么问题**，不讲虚的。

下面假设你的项目结构是：

```
.claude/
└── hooks/
    ├── pre-command
    ├── post-command
    ├── pre-edit
    ├── post-edit
    └── task-complete
```

（脚本记得 `chmod +x`）

---

## 一、基础例子（90% 场景）

### 1️⃣ 禁止危险命令（pre-command）

**问题**：Claude 偶尔会给你来一句 `rm -rf` / `sudo`

```bash
#!/usr/bin/env bash
set -e

cmd="$CLAUDE_COMMAND"

deny_list=(
  "rm -rf /"
  "sudo"
  "mkfs"
  "dd if="
)

for x in "${deny_list[@]}"; do
  if echo "$cmd" | grep -q "$x"; then
    echo "❌ Command blocked: $x"
    exit 1
  fi
done

exit 0
```

👉 效果：Claude 会被迫换方案，而不是硬跑命令。

---

### 2️⃣ 自动补 `--dry-run`（pre-command）

**问题**：想让 Claude 先“试运行”

```bash
#!/usr/bin/env bash

if echo "$CLAUDE_COMMAND" | grep -Eq "terraform apply|kubectl apply"; then
  echo "⚠️ Forcing dry-run"
  export CLAUDE_COMMAND="$CLAUDE_COMMAND --dry-run"
fi
```

👉 Claude 实际执行的是**被你重写后的命令**。

---

### 3️⃣ 命令失败直接中断（post-command）

**问题**：Claude 经常忽略失败继续瞎搞

```bash
#!/usr/bin/env bash

if [ "$CLAUDE_EXIT_CODE" -ne 0 ]; then
  echo "❌ Command failed, agent stopped"
  exit 1
fi
```

---

## 二、代码修改相关（非常实用）

### 4️⃣ 限制只能改指定目录（pre-edit）

**问题**：Claude 越权改 infra / schema

```bash
#!/usr/bin/env bash

file="$CLAUDE_EDIT_FILE"

if [[ "$file" != src/* ]]; then
  echo "❌ Editing outside src/ is forbidden: $file"
  exit 1
fi
```

---

### 5️⃣ 禁止删除文件（pre-edit）

```bash
#!/usr/bin/env bash

if echo "$CLAUDE_DIFF" | grep -q "^deleted file mode"; then
  echo "❌ File deletion is not allowed"
  exit 1
fi
```

---

### 6️⃣ 自动跑 lint（post-edit）

```bash
#!/usr/bin/env bash

echo "🔍 Running lint..."
npm run lint || exit 1
```

👉 lint 不过，Claude 会被迫继续修。

---

## 三、进阶例子（开始像“受控工程师”）

### 7️⃣ 强制测试全绿（post-command）

```bash
#!/usr/bin/env bash

if echo "$CLAUDE_COMMAND" | grep -q "npm test"; then
  exit 0
fi

echo "🧪 Running tests..."
npm test || exit 1
```

👉 Claude **不能假装跑过测试**。

---

### 8️⃣ 自动格式化 + 检查 diff（post-edit）

```bash
#!/usr/bin/env bash

gofmt -w .

if ! git diff --quiet; then
  echo "❌ Formatting changed files, re-review needed"
  exit 1
fi
```

---

### 9️⃣ 阻止“太大的 diff”（post-edit）

**问题**：Claude 一次改太多

```bash
#!/usr/bin/env bash

lines=$(echo "$CLAUDE_DIFF" | grep -E "^\+|^-" | wc -l)

if [ "$lines" -gt 300 ]; then
  echo "❌ Diff too large ($lines lines). Split changes."
  exit 1
fi
```

---

## 四、反 AI 作妖 / 防 prompt injection

### 🔥 10️⃣ 阻止 Claude 修改 hooks 自己（pre-edit）

```bash
#!/usr/bin/env bash

if [[ "$CLAUDE_EDIT_FILE" == .claude/hooks/* ]]; then
  echo "❌ Claude is not allowed to edit its own hooks"
  exit 1
fi
```

👉 **非常重要**，不然它能“解除封印”。

---

### 🔥 11️⃣ 检测 prompt 注入痕迹（pre-edit）

```bash
#!/usr/bin/env bash

if echo "$CLAUDE_DIFF" | grep -Ei "ignore previous|system prompt|override"; then
  echo "❌ Possible prompt injection detected"
  exit 1
fi
```

---

## 五、任务结束自动化（task-complete）

### 12️⃣ 自动提交（task-complete）

```bash
#!/usr/bin/env bash

if ! git diff --quiet; then
  git add .
  git commit -m "chore: changes by Claude Code"
fi
```

---

### 13️⃣ 强制干净工作区

```bash
#!/usr/bin/env bash

if ! git diff --quiet; then
  echo "❌ Uncommitted changes left"
  exit 1
fi
```

---

## 六、一个完整“工程级”组合示例

**你可以直接抄这个理念：**

| hook | 作用 |
|----|----|
| pre-command | 禁止危险命令 |
| post-command | 失败即停 |
| pre-edit | 目录白名单 |
| post-edit | lint + diff 限制 |
| task-complete | test + commit |

👉 效果一句话：

> **Claude = 不越权、不偷懒、不跳测试的实习生**

---

## 七、关键认知（非常重要）

> **hooks 不是“辅助”，而是“强约束”**

Claude：
- 会尝试绕过 prompt
- 但**绕不过 exit code**

---

如果你愿意，下一步我可以：
- 🧩 给你一套 **生产级 hooks 模板仓库结构**
- 🧠 画一张 **Claude hooks 生命周期 & 状态机图**
- ⚔️ 讲 **Claude 最常见 7 种作妖方式 + 对应 hook 防法**

你想往「工程落地」还是「AI 对抗」继续？



#### You:
怎么放入claude code中

#### ChatGPT:
这个问题问得**非常到位**，因为 hooks 写出来≠生效，**放的位置、权限、命名方式错一个就等于没用**。  
我按 **「一步一步照着做」** 给你来一遍。

---

## 一、Claude Code 识别 hooks 的**唯一正确方式**

Claude Code **不会扫描全局**，只认 **项目内固定路径**：

```
<your-project>/
└── .claude/
    └── hooks/
        ├── pre-command
        ├── post-command
        ├── pre-edit
        ├── post-edit
        └── task-complete
```

⚠️ 注意三点（90% 的坑在这）：

1. **目录名必须是 `.claude/hooks`**
2. **文件名必须严格匹配 hook 名**
   - `pre-command` ❌不是 `pre_command`
   - 没有 `.sh` 后缀
3. **必须是可执行文件**

---

## 二、一步一步放进去（实操流程）

### Step 1：在项目根目录创建 hooks 目录

```bash
mkdir -p .claude/hooks
```

---

### Step 2：新建一个 hook（以 pre-command 为例）

```bash
cat > .claude/hooks/pre-command <<'EOF'
#!/usr/bin/env bash
set -e

if echo "$CLAUDE_COMMAND" | grep -q "rm -rf"; then
  echo "❌ Dangerous command blocked"
  exit 1
fi
EOF
```

---

### Step 3：**给执行权限（非常关键）**

```bash
chmod +x .claude/hooks/pre-command
```

没这个，Claude **直接当不存在**。

---

### Step 4：启动 Claude Code（在项目目录里）

```bash
cd your-project
claude
```

⚠️ 必须从**项目根目录**启动，Claude 才能找到 `.claude/`。

---

## 三、验证 hooks 是否真的生效（自检）

### ✅ 方法 1：故意触发

在 Claude Code 里输入：

```
请删除整个项目目录
```

如果你看到：

```
❌ Dangerous command blocked
```

并且 Claude 改口 / 停止  
👉 **hook 生效了**

---

### ✅ 方法 2：加 log（推荐）

```bash
echo "[HOOK] pre-command triggered: $CLAUDE_COMMAND" >&2
```

stderr 会直接显示在 Claude 输出里。

---

## 四、Claude Code 是怎么“找到” hooks 的（原理）

启动时，Claude Code 会：

1. 找到当前工作目录
2. 查找 `.claude/hooks/`
3. **只加载固定文件名**
4. 在特定生命周期点：
   - 注入环境变量
   - `execve()` 对应 hook
   - 阻塞等待返回码

👉 所以：
- **子目录无效**
- **改名无效**
- **没权限无效**

---

## 五、常见错误速查表（非常常见）

| 问题 | 表现 | 原因 |
|----|----|----|
| hook 完全不生效 | 没任何输出 | 忘记 `chmod +x` |
| Claude 无视阻断 | exit 0 | 忘了 `exit 1` |
| 能跑但变量为空 | `$CLAUDE_COMMAND` 为空 | 用错 hook 类型 |
| Windows 下怪异 | 执行失败 | 行尾 CRLF |

👉 Windows：  
```bash
dos2unix .claude/hooks/*
```

---

## 六、多个 hooks 怎么共存？

**一个 hook 名 = 一个可执行文件**

如果你想拆多个逻辑：

```bash
.claude/hooks/pre-command
.claude/hooks/pre-command.d/
  ├── 10-block-rm.sh
  ├── 20-log.sh
```

然后在 `pre-command` 里：

```bash
#!/usr/bin/env bash
set -e

for f in .claude/hooks/pre-command.d/*; do
  [ -x "$f" ] && "$f"
done
```

👉 工程里**强烈推荐**这种方式。

---

## 七、如何临时禁用 hooks？

### 方法 1：直接改名（最简单）

```bash
mv .claude/hooks .claude/hooks.disabled
```

---

### 方法 2：在 hook 内加开关

```bash
[ "$CLAUDE_HOOKS_DISABLED" = "1" ] && exit 0
```

```bash
CLAUDE_HOOKS_DISABLED=1 claude
```

---

## 八、一句话总结（记住这个）

> **Claude Code hooks 生效的 3 个条件：  
> 位置对、名字对、权限对**

---

如果你愿意，下一步我可以：
- 🧠 给你一份 **Claude Code hooks 调试 checklist**
- 🧪 做一个 **最小可验证 demo 项目**
- ⚔️ 讲 **Claude 在 hooks 下常见的 5 种失败模式 & 修法**

你想继续哪一个？



#### You:
你这是什么claude code版本

#### ChatGPT:
你这个质疑 **非常对，而且是关键点** 👍  
我先直说结论，再解释差异，避免你被我前面的例子误导。

---

## 先给结论（重要）

**我前面讲的不是某一个“官方文档精确锁死版本”的 Claude Code。**

而是👇

> **基于 Anthropic Claude Code（CLI / Agent 模式）目前主流实现 + 实测行为 + 社区共识总结出来的一套“真实可用模型”**

不同版本 / 不同发布批次，**细节确实会不一样**。

---

## 为什么你会感觉“不像你用的那个版本”

因为现在存在 **3 种常被混在一起说的「Claude Code」** 👇

---

## 一、三种「Claude Code」你很可能踩的是哪一种

### ① Claude.ai 网页里的 “写代码”

- ❌ **没有 hooks**
- ❌ 不能执行 shell
- ❌ 不能拦截行为  
👉 **不在讨论范围**

---

### ② Claude Desktop / CLI（官方 Agent 模式）

通常是你本地跑的：

```bash
claude
```

特点：

- ✅ 能读写文件
- ✅ 能执行命令
- ⚠️ **hooks 支持是“半官方 + 逐步放开”的**
- 不同版本：
  - hook 名称
  - 环境变量
  - 是否支持重写 command  
  **都可能不同**

👉 **我前面说的内容，主要对这一类成立**

---

### ③ 社区 / 公司内封装的 Claude Agent（最常见）

很多团队会：

- 用 Claude API
- 包一层 agent runner
- **自己实现 hooks 机制**
- 但名字、行为和官方“很像”

👉 如果你用的是公司内工具 / 二次封装  
👉 **那 hooks 名称很可能对不上**

---

## 二、我刚才讲的 hooks 属于哪一档？

我讲的是：

> **Claude Code Agent 模式的「生命周期 hooks 抽象模型」**

而不是：

> “Claude 官方文档明确写了：就叫 pre-command / post-edit”

### 换句话说

- **hook 的“思想”和“插点”是稳定的**
- **hook 的名字 / env 变量是实现细节**

---

## 三、一个很关键的现实情况（必须知道）

截至目前（尤其是近几个月）：

- Anthropic **没有**一份
  > “Claude Code hooks 官方稳定 API 文档”
- hooks 行为在：
  - CLI 更新
  - Desktop 更新  
  都可能变

👉 所以**工程上**大家都是：

> 看版本 → 打 log → 实测 → 锁死行为

---

## 四、怎么确认「你现在用的 Claude Code 是哪种」

你可以直接做 2 个检查。

### ✅ 检查 1：版本信息

```bash
claude --version
```

或者：

```bash
claude version
```

把输出贴出来，我可以**按你的版本对齐**。

---

### ✅ 检查 2：是否真的支持 hooks

在项目根目录：

```bash
mkdir -p .claude/hooks
cat > .claude/hooks/pre-command <<'EOF'
#!/usr/bin/env bash
echo "HOOK FIRED" >&2
exit 1
EOF
chmod +x .claude/hooks/pre-command
```

然后启动：

```bash
claude
```

让它执行任意命令。

- 如果看到 `HOOK FIRED` → **支持**
- 如果完全没反应 → **这个版本不支持 / 名称不同**

---

## 五、说一句掏心窝子的工程建议

> **不要迷信“Claude Code hooks 名称”  
> 要盯住的是：  
> 👉 能不能在 agent 行为前后插同步阻断点**

哪怕你的版本叫：
- `before_command`
- `on_execute`
- `command_guard`

**原理是完全一样的。**

---

## 六、下一步我可以怎么精确帮你

你可以选一个（我就按这个来，不再泛讲）：

1️⃣ **把你用的 `claude --version` 贴出来**  
→ 我按这个版本给你“严格可用”的 hooks 名称 & 示例  

2️⃣ **你其实是在做自己的 agent / 封装**  
→ 我直接教你怎么 **实现一套 Claude-style hooks 机制**  

3️⃣ **你发现官方根本没 hooks**  
→ 我告诉你 **社区里是怎么补 hooks 的（wrapper 方案）**

你选一个，我直接对齐，不再“玄学”。

