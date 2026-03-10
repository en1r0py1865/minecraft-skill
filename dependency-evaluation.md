# OpenClaw 跨 Skill 依赖机制：minecraft-bridge 评估报告

---

## 核心发现：OpenClaw 无原生依赖系统

OpenClaw 的 Skill 架构基于**独立触发**原则：

```
用户消息 → 语义匹配 → 单个 Skill 触发 → 执行
```

框架**不会**：
- 自动检测其他 Skill 是否安装
- 声明 Skill 间的依赖关系
- 在 Skill A 触发时自动加载 Skill B 的内容

这意味着 `minecraft-bridge` 作为"底层服务层"时，框架本身提供零保障。

---

## 四种依赖场景分析

### 场景 1：纯知识 Skill（不需要 bridge）

**代表**: `minecraft-wiki`

```
用户: "钻石在哪个Y层最多？"
触发: minecraft-wiki ✅
Bridge状态: 完全无关
行为: 直接从 references/ores.md 回答
结论: ✅ 完美工作，无依赖问题
```

**设计决策**: Wiki 类 Skill 不应该触碰 bridge，这样在游戏未运行时也能提供价值。

---

### 场景 2：同一渠道内的顺序调用

**代表**: 用户在一个对话中先问 wiki 再问操控

```
用户: "钻石在哪层？"
→ minecraft-wiki 触发，回答 Y=-58

用户: "好，帮我挖钻石"
→ minecraft-bridge 触发，调用 /mine diamond_ore
```

**OpenClaw 实际行为**:
- 每条消息独立触发 Skill
- 不同 Skill 可以在同一对话中分别触发
- **关键问题**: 如果 bridge 服务未启动，第二条消息会触发 bridge Skill，
  但 `GET /status` 会返回连接拒绝 → Skill 的错误处理逻辑会提示用户

**结论**: ✅ 顺序调用正常工作，依靠 Skill 内部的健康检查而非框架

---

### 场景 3：Skill 需要调用 bridge API（混合 Skill）

**代表**: `minecraft-planner`（规划 Skill，既要生成步骤计划，又要读取背包状态）

```
用户: "帮我规划获得钻石镐的步骤"
→ minecraft-planner 触发
→ Skill 内部需要调用 GET /inventory 获取当前背包
→ 但 bridge 可能未运行
```

**OpenClaw 实际执行路径**:

```
minecraft-planner SKILL.md 触发
    ↓
Skill 指令: "调用 GET /inventory 检查背包"
    ↓
[情况A] Bridge 运行中:
    curl http://localhost:3001/inventory → 成功
    → 规划时标注哪些材料已有/缺少
    → ✅ 最佳体验

[情况B] Bridge 未运行:
    curl → ECONNREFUSED
    → Skill 指令: "如果bridge未运行，基于用户描述的状态规划"
    → Agent 切换到"询问用户当前资源状态"模式
    → ⚠️ 降级工作，体验下降但不崩溃
```

**关键设计模式**: Skill 必须**显式处理 bridge 不可用**的降级逻辑，
而不能假设 bridge 一定在运行。这是与微服务完全不同的思维。

---

### 场景 4：多 Skill 协同执行（最复杂）

**代表**: "自动化采矿"流程

```
理想流程:
minecraft-planner → 生成计划
    ↓
minecraft-bridge → 执行移动
    ↓
minecraft-bridge → 执行挖掘
    ↓
minecraft-craft → 检查是否可合成
```

**OpenClaw 实际行为** (关键洞察):

OpenClaw 的 Agent 在单次推理中**可以顺序调用多个工具/脚本**。
由于 `minecraft-bridge` 暴露为 HTTP API，Agent 可以：

```
Agent 推理:
1. 读取 minecraft-planner/SKILL.md（触发的 Skill）
2. 内部决策：我需要先查背包状态
3. 执行 shell: curl http://localhost:3001/inventory
4. 解析结果，生成计划
5. 执行 shell: curl -X POST .../move -d '{"x":...}'
6. 等待完成，执行 curl .../mine ...
```

**实测结论**: 
- ✅ 单 Agent 内可以多次调用 bridge API（HTTP 调用不需要"触发另一个 Skill"）
- ⚠️ 但每次 HTTP 调用都需要 Skill 指令明确指定
- ❌ 无法"透明地"在 Skill 间传递状态（除非通过 OpenClaw Memory 文件）

---

## 依赖架构的三个核心问题与解法

### 问题 1：Skill 无法感知其他 Skill 是否安装

**问题**: `minecraft-planner` 无法知道用户是否安装了 `minecraft-bridge`

**解法（已实现）**:
```markdown
# 在 minecraft-planner 的 SKILL.md 中加入：

## 前置检查
执行前先检查 bridge 状态：
  curl -sf http://localhost:${MC_BRIDGE_PORT:-3001}/status

- 成功且 connected=true → 调用实时数据 API
- 连接拒绝 → 进入"离线规划模式"：向用户询问当前资源
- connected=false → 提示用户打开 Minecraft 并等待 Bot 连接
```

**效果**: Skill 在没有 bridge 时优雅降级，而不是崩溃。

---

### 问题 2：Bridge 服务状态在 Agent 重启后消失

**问题**: `bridge-server.js` 是独立进程，Agent 重启不影响它，
但 OpenClaw Memory 可能没记录"bridge 正在运行"。

**解法**:
```markdown
# bridge-server.js 启动时写入状态文件
echo '{"running":true,"port":3001,"started":"2026-03-08T10:00:00Z"}' \
  > ~/.openclaw/minecraft-bridge.state

# 其他 Skill 检查这个文件，比 TCP 连接更快
```

**已在 `start.sh` 中实现**: 用 PID 文件记录运行状态，`stop.sh` 清理。

---

### 问题 3：OpenClaw 的 Progressive Disclosure 与 bridge 依赖冲突

**问题**: OpenClaw 的 Skill 描述（~100 词）需要说明触发条件，
但同时需要说明"需要 bridge"，这会消耗宝贵的 description 空间。

**实测最优写法**:
```yaml
# ❌ 错误：在 description 中说明依赖
description: >
  Requires minecraft-bridge to be running. Plans tasks for Minecraft...

# ✅ 正确：description 只写触发条件，依赖写在 SKILL.md body 中
description: >
  Plan step-by-step Minecraft tasks. Use when user asks 'plan my minecraft
  goals', 'how to get diamond pickaxe', 'what should I do next in minecraft'.
  
# SKILL.md body 开头：
## Prerequisites
Check bridge status before any live-game operation: ...
```

**原理**: description 是触发过滤器，body 是执行指南。依赖检查属于执行层，
不应污染触发层。

---

## 最终架构评估：两类 Skill 的正确定位

```
┌─────────────────────────────────────────────────────────┐
│               TIER 1: 知识/规划 Skill                     │
│   minecraft-wiki, minecraft-farm, minecraft-speedrun     │
│                                                          │
│   • 完全离线工作                                           │
│   • Bridge = 可选增强（能查背包则更准确）                   │
│   • 触发无门槛，受众最广                                   │
└─────────────────────────────────────────────────────────┘
                          ↕ 可选增强
┌─────────────────────────────────────────────────────────┐
│               TIER 2: 操控 Skill（Bridge 核心）            │
│   minecraft-bridge（基础服务）                            │
│   minecraft-planner（规划+执行）                          │
│   minecraft-craft（合成+执行）                            │
│                                                          │
│   • 需要 Bridge 运行才能执行                               │
│   • 内置优雅降级：Bridge 不可用时切换到询问模式             │
│   • 依赖检查逻辑写在 SKILL.md body，不写在 description     │
└─────────────────────────────────────────────────────────┘
                          ↕ 独立协议
┌─────────────────────────────────────────────────────────┐
│               TIER 3: 管理 Skill（RCON 独立）              │
│   minecraft-server-admin                                 │
│                                                          │
│   • 不依赖 minecraft-bridge，直接连 RCON                   │
│   • 不依赖游戏客户端，服务器管理员专用                      │
│   • 完全独立的认证体系（MC_RCON_PASSWORD）                  │
└─────────────────────────────────────────────────────────┘
```

---

## OpenClaw 跨 Skill 依赖的最佳实践总结

| 原则 | 说明 |
|------|------|
| **不假设服务运行** | 每次调用 bridge 前先做健康检查 |
| **优雅降级** | Bridge 不可用 = 切换到询问模式，不是报错退出 |
| **状态写入 Memory** | 用 OpenClaw Memory 在 Agent 重启间传递游戏状态 |
| **description 只写触发** | 依赖声明放在 SKILL.md body，不污染 description |
| **知识 Skill 不耦合** | wiki/farm/speedrun 不依赖 bridge，受众更广 |
| **独立协议独立 Skill** | RCON 和 Mineflayer 是不同协议，不共享 Skill |
