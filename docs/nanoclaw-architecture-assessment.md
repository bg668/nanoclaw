# NanoClaw 架构分析报告

## 目标架构模型映射

### 核心模块对标结果

#### 1. 通政司（Inbound Parse/Validate/Normalize）
**目标**：inbound parse/validate/normalize -> 仅输出 InboundEvent

**实现文件**：
- `src/channels/registry.ts` - Channel 注册工厂模式
- `src/channels/index.ts` - Channel 自注册
- `src/types.ts` - Channel 接口、NewMessage 定义
- `src/router.ts` - 格式化消息（第13-25行）

**核心函数**：
- `Channel.sendMessage()` - 输出接口
- `OnInboundMessage` - inbound 回调类型（第96行）
- `formatMessages()` - 规范化消息格式

**匹配度**：**部分匹配**
- ✅ 有统一的消息输入规范化（formatMessages）
- ✅ 通过 Channel 接口实现多渠道输入解析
- ❌ 缺少“已校验输入契约”的显式 InboundEvent 定义（目前 `NewMessage` 同时承担传输、存储与已验证事件语义，验证边界不清晰；见“优先级 2”）
- ❌ 验证逻辑散落在 `index.ts` 的消息接收与处理路径中（如 sender allowlist 检查），不集中

---

#### 2. 中书省（Orchestration/Session Routing/Top-level Reasoning）
**目标**：编排/会话路由/顶层推理

**实现文件**：
- `src/index.ts` - 主编排器（核心）
  - `startMessageLoop()` - 消息循环（第349-447行）
  - `processGroupMessages()` - 按组处理（第151-266行）
  - `runAgent()` - Agent 调用（第268-347行）
- `src/group-queue.ts` - 按组队列管理
- `src/task-scheduler.ts` - 任务调度

**核心数据流**：
```
getNewMessages() → messagesByGroup → queue.enqueueMessageCheck()
                                  → processGroupMessages()
                                  → runAgent()
```

**匹配度**：**完全匹配**
- ✅ 完整的消息循环与编排（index.ts）
- ✅ 会话管理 `sessions` 字典（index.ts 第68行）
- ✅ 按组路由（processGroupMessages）
- ✅ 顶层决策逻辑清晰（trigger 检查、isMain 判断）

---

#### 3. 翰林院/史馆（Retrieval/Memory/History with Gated Writes）
**目标**：检索/内存/历史上下文，仅允许受控的长期写入

**实现文件**：
- `src/db.ts` - SQLite 消息与会话存储（第1-100行+）
- `groups/*/CLAUDE.md` - 按组内存文件
- `groups/global/CLAUDE.md` - 全局内存（仅 main 可写）
- `src/container-runner.ts` - 内存挂载（第59-114行）
  - 全局目录 read-only for non-main（第106-113行）
  - 按组会话隔离（第116-147行）

**DB 表结构**（db.ts）：
- `messages` - 消息历史
- `chats` - 对话元数据
- `sessions` - 会话 ID 映射
- `router_state` - 状态游标

**写入控制**：
- `setRegisteredGroup()` 仅从 main 调用（ipc.ts 第435-443行）
- `storeMessage()` 有发件人白名单检查（index.ts 第552-566行）

**匹配度**：**完全匹配**
- ✅ 完整的消息历史检索（getMessagesSince, formatMessages）
- ✅ 多层内存系统（全局 + 按组 CLAUDE.md）
- ✅ 写入受控（isMain 检查贯穿 ipc.ts）
- ✅ 会话隔离（每组独立 .claude/ 目录）

---

#### 4. 六部（Execution Backend/Tool Registry/Container Lifecycle/Artifact）
**目标**：执行后端/工具注册/容器生命周期/构件处理

**实现文件**：
- `src/container-runner.ts` - 主容器执行器
  - `runContainerAgent()` - 启动容器（第267-400+行）
  - `buildVolumeMounts()` - 挂载配置（第59-212行）
  - `buildContainerArgs()` - 容器参数（第215-265行）
- `src/container-runtime.ts` - 容器运行时管理
- `container/agent-runner/src/index.ts` - 容器内 agent 入口
- `src/credential-proxy.ts` - 凭证隔离与代理

**工具注册**：
- 通过 Agent SDK 内置工具（Bash、Read、Write、WebSearch）
- MCP 服务器通过 stdio（ipc-mcp-stdio.ts）

**容器生命周期**：
```
spawn(CONTAINER_RUNTIME_BIN) → mount volumes → write stdin → 
parse OUTPUT_START/END markers → streaming output → container.wait()
```

**制件处理**：
- 日志：`groups/{folder}/logs/container-*.log`（第307-308行）
- IPC 输出：`/workspace/ipc/messages/*.json`（ipc.ts）

**匹配度**：**完全匹配**
- ✅ 完整的容器生命周期管理
- ✅ 动态挂载配置（additionalMounts 验证）
- ✅ 凭证代理隔离（credential-proxy.ts）
- ✅ 制件收集（日志、IPC 消息）

---

#### 5. 门下省（Outbound Reply Validation/Guard Approval/Filter/Reject）
**目标**：出站回复验证/守卫批准/过滤/拒绝，不生成内容

说明：这里应区分两层验证——技术层（路由可达性）与策略层（目标许可、内容合规）。

**实现文件**：
- `src/router.ts` - 出站路由与格式化
  - `stripInternalTags()` - 移除内部标记（第27-29行）
  - `formatOutbound()` - 出站格式化（第31-35行）
  - `routeOutbound()` - 路由到 channel（第37-45行）
- `src/sender-allowlist.ts` - 发件人白名单
  - `isSenderAllowed()` - 发件人检查
  - `shouldDropMessage()` - 消息过滤决策
- `index.ts` 中的出站调用（第226行、第614行）

**验证逻辑**：
- `formatOutbound()` 仅移除内部标记，不修改内容
- `routeOutbound()` 仅验证“技术路由可达性”（channel 所有权/连接状态）后发送
- 发件人检查位于 inbound（合理）；但 outbound 缺少独立策略守卫（设计缺口）

**匹配度**：**部分匹配**
- ✅ 有出站格式化与路由逻辑
- ✅ 移除内部推理标记（_internal_）
- ❌ 缺少出站内容守卫/批准机制（仅有格式化，无拒绝逻辑）
- ❌ 出站路径缺少策略层守卫（目标 jid 许可、内容策略），当前仅有技术层路由校验

---

### 支持模块对标

#### Data Schema（数据架构）
**实现**：
- `src/types.ts` - 完整的 TypeScript 接口定义
  - `NewMessage`、`RegisteredGroup`、`ScheduledTask`、`Channel`
- `src/db.ts` - SQLite Schema（第18-95行）

**匹配度**：**完全匹配**

---

#### Role/Assets（角色/资产）
**实现**：
- `isMain` 标志（RegisteredGroup.isMain，types.ts 第42行）
  - 在 ipc.ts 中贯穿权限检查（第204-210, 279-288, 315-327等行）
- SOUL（Session）：`sessions` 字典与 .claude/ 目录
- ROLE（权限）：isMain 二元角色模型
- MEMORY：CLAUDE.md 文件 + SQLite messages 表
- POLICY：sender-allowlist.json 外部配置

**匹配度**：**部分匹配**
- ✅ 会话隔离（SOUL）
- ✅ 权限角色（ROLE：main vs non-main）
- ✅ 内存系统（MEMORY：全局 + 按组）
- ❌ 缺少细粒度 POLICY（仅有二元 isMain，无 RBAC）

---

#### Archive/State（档案/状态）
**实现**：
- `store/messages.db` - SQLite 持久化
- `data/sessions/` - 会话档案
- `data/ipc/` - IPC 命令档案
- `src/config.ts` - 配置管理

**匹配度**：**完全匹配**

---

#### Capability Adapters/Providers/Config/Skills
**实现**：
- Channel 注册工厂（channels/registry.ts）- Adapter
- Credential Proxy（credential-proxy.ts）- Provider
- Config（config.ts）- Configuration
- Skills（.claude/skills/）- Capability packages

**匹配度**：**完全匹配**

---

## 架构间隙与耦合问题

### 🔴 Critical Gaps

#### 1. 出站守卫缺失（门下省不完整）
**位置**：`src/router.ts`、`src/index.ts` 第614行、226行

**问题**：
```typescript
// router.ts - 仅做格式化，无拒绝逻辑
export function formatOutbound(rawText: string): string {
  const text = stripInternalTags(rawText);
  if (!text) return '';
  return text;  // 直接返回，无守卫
}

// index.ts:614 - 出站无权限检查
const text = formatOutbound(rawText);
if (text) await channel.sendMessage(jid, text);
```

**风险**：除 IPC 命令路径外，主流程的直接出站发送缺少同等策略守卫，导致控制点不一致

**证据**：ipc.ts 第76-83行有 isMain 权限检查，但出站路径无此机制

---

#### 2. 通政司验证过度分散
**位置**：`src/index.ts` 第552-566行（allowlist 检查）、第172-180行（trigger 检查）

**问题**：
```typescript
// index.ts:552 - 消息已存入 DB 后才验证
if (!msg.is_from_me && !msg.is_bot_message && registeredGroups[chatJid]) {
  const cfg = loadSenderAllowlist();
  if (shouldDropMessage(chatJid, cfg) && 
      !isSenderAllowed(chatJid, msg.sender, cfg)) {
    return;  // 已存入 DB，仅跳过处理
  }
}
```

**风险**：
- 被拒绝的消息仍在数据库中（信息泄露）
- 验证逻辑不中心化
- Trigger 检查与 allowlist 检查分离（第172-180 vs 552-566）

---

#### 3. InboundEvent 定义缺失
**问题**：使用 `NewMessage` 作为内部类型，但无显式的规范化 InboundEvent 结构

**风险**：channel 可能传入超出规范的数据，验证不明确

---

### 🟡 Moderate Coupling Issues

#### 4. 容器与 IPC 的权限混合
**位置**：`src/ipc.ts` 与 `src/container-runner.ts`

**问题**：
- IPC 消息文件由 container 写入、host 读取，权限判断在 host（ipc.ts）
- 但 container 可直接向 /workspace/ipc 写任意内容
- 权限检查延后到 host 处理（防守后置）

**风险**：如果 container 逃逸，可写入恶意 IPC 文件后续被执行

---

#### 5. 会话与内存的紧耦合
**位置**：`src/index.ts` 第306-308行、container-runner.ts 第116-147行

**问题**：
```typescript
// 会话 ID 存入 DB，但 .claude/ 目录单向挂载
// 没有显式的一致性模型
if (output.newSessionId) {
  sessions[group.folder] = output.newSessionId;
  setSession(group.folder, output.newSessionId);  // DB 存储
}
```

**风险**：
- 内存恢复机制隐含（依赖 .claude/ 挂载点）
- 无显式的内存版本管理或 rollback 机制
- 如果 session 文件与 DB 不同步，无恢复策略

---

### 🟢 Minor Issues

#### 6. 容器配置验证不完整
**位置**：`src/mount-security.ts`

**问题**：additionalMounts 验证基于外部文件 (mount-allowlist.json)，但无内联防御

**风险**：如果外部配置被篡改（尽管无法从容器访问），所有检查失效

---

## 细致的匹配度评分

| 模块 | 完全度 | 说明 |
|------|--------|------|
| 通政司 | 部分匹配 | 缺 InboundEvent 定义、验证分散 |
| 中书省 | 完全匹配 | 核心编排逻辑完整 |
| 翰林院/史馆 | 完全匹配 | 内存+会话隔离清晰 |
| 六部 | 完全匹配 | 容器生命周期完整 |
| 门下省 | 部分匹配 | 出站无守卫、权限检查缺失 |

---

## 推荐（文档更新方向，最小改动）

### 优先级 1：出站守卫补齐（门下省）

**建议**：添加 `formatOutbound()` 后的权限验证层

```markdown
## 出站守卫（门下省）

### 当前实现
src/router.ts 的 formatOutbound() 仅移除内部标记。

### 建议改进（文档）
建立显式的 "outbound guard" 步骤：
1. 格式化（现有）
2. 守卫检查（缺失）- 重点验证“目标 jid 是否在允许范围”与“内容是否满足策略”
3. 内容校验（建议）- 可选的内容过滤规则
4. 发送（现有）

关键：权限检查不应散落在调用点；建议新增独立 `outbound-guard`（或在中书省统一调用），
再由 `router.ts` 专注执行技术路由。
```

### 优先级 2：规范化 InboundEvent（通政司）

**建议**：在 types.ts 中显式定义

```typescript
// types.ts（建议）
// 统一的规范化输入（保持与现有 NewMessage 一致的 snake_case）
export interface InboundEvent {
  id: string;
  chat_jid: string;
  sender: string;
  sender_name: string;
  content: string;
  timestamp: string;
  is_from_me: boolean;
  is_bot_message?: boolean;
}

// 与当前 NewMessage 的关系：InboundEvent 表达“已完成校验的输入契约”，
// NewMessage 保持持久化/传输模型，二者字段命名保持一致可降低迁移成本。
```

### 优先级 3：权限模型文档化（Role/Assets）

**建议**：在 SPEC.md 中明确权限矩阵

```markdown
## 权限模型

### 角色与权限

| 操作 | Main | Non-Main |
|------|------|----------|
| 读全局内存 | ✅ | ✅ (read-only) |
| 写全局内存 | ✅ | ❌ |
| 列所有组 | ✅ | ❌ |
| 跨组发消息 | ✅ | ❌ |
| 本组发消息 | ✅ | ✅ |
| 注册新组 | ✅ | ❌ |

实现位置：
- 读写控制：ipc.ts (schedule_task, register_group)
- 内存挂载：container-runner.ts (buildVolumeMounts)
```

### 优先级 4：内存一致性说明（翰林院/史馆）

**建议**：在 SPEC.md 中记录会话恢复流程

```markdown
## 会话恢复

### 流程
1. Main process 启动，读 DB 中 sessions 表
2. 为每组准备 data/sessions/{folder}/.claude/
3. 容器启动时挂载 .claude/ 目录
4. Agent SDK 从 .claude/JSONL 恢复会话
5. 新 sessionId 写回 DB (setSession)

### 保证
- .claude/ 目录是 single source of truth
- DB 中的 session_id 用于恢复指针
- 无显式事务，当前流程以“先目录恢复、后写 DB 指针”为主
```

### 优先级 5：IPC 权限防守前置（六部/中书省）

**建议**：在设计文档中强调 container 信任模型

```markdown
## 容器信任边界

### IPC 安全模型
- ❌ 不信任 container（假设其可能写入恶意 IPC 文件）
- ✅ 信任 host IPC 处理器验证每个请求
- ✅ 信任外部配置文件（mount-allowlist.json）无法从容器访问

### 实施
ipc.ts processTaskIpc() 中的所有权限检查是 host 侧防守，
不依赖 container 的"好行为"。
```

---

## 总结

| 维度 | 评分 | 说明 |
|------|------|------|
| 整体架构清晰度 | ⭐⭐⭐⭐ | 代码结构与文档一致，易于理解 |
| 安全隔离 | ⭐⭐⭐⭐ | 容器隔离、权限模型清晰 |
| 权限一致性 | ⭐⭐⭐ | isMain 二元模型简洁，但出站守卫缺失 |
| 内存管理 | ⭐⭐⭐⭐ | 多层隔离设计完善 |
| 扩展性 | ⭐⭐⭐⭐ | Skill 系统与 channel 工厂模式良好 |

**核心建议**：完善出站守卫、规范化 InboundEvent、文档化权限矩阵。改动范围最小，价值最大。
