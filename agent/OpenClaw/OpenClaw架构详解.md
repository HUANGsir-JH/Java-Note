# OpenClaw 架构详解

> ⚠️ **注意**：这里讲解的是由 **Peter Steinberger** 创建的开源自托管 AI 助手框架 OpenClaw（原称 Clawdbot），而非科学计算领域的同名项目。

---

## 🏗️ 整体架构概览

OpenClaw 采用 **Hub-and-Spoke（中心辐射）架构**，核心是一个单进程的 **Gateway（网关）** 作为控制平面，连接各种消息渠道与 AI Agent 运行时 。

```text
┌─────────────────────────────────────────┐
│           用户交互层 (Channels)          │
│  WhatsApp │ Telegram │ Discord │ iMessage│
└────────────┬────────┬────────┬──────────┘
             │        │        │
             ▼        ▼        ▼
┌─────────────────────────────────────────┐
│         Gateway 控制平面 (Hub)          │
│  • WebSocket 服务器 (默认 127.0.0.1:18789)│
│  • 消息路由 & 访问控制                    │
│  • 会话管理 & 状态持久化                  │
│  • 插件加载 & 工具调度                    │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│         Agent Runtime (执行引擎)         │
│  • 会话解析 & 上下文组装                  │
│  • LLM 模型调用 (Anthropic/OpenAI等)      │
│  • 工具执行 (bash/浏览器/文件操作等)      │
│  • 流式响应 & 状态回写                    │
└─────────────────────────────────────────┘
```

---

## 🔑 四大核心组件

### 1️⃣ Channel Adapters（渠道适配器）

每个消息平台都有独立的适配器，实现统一接口 ：

| 职责  | 说明  |
| --- | --- |
| **认证** | WhatsApp用QR配对、Telegram用Bot Token、iMessage需macOS原生集成 |
| **入站解析** | 提取文本/媒体/表情/回复上下文，统一格式 |
| **访问控制** | Allowlist白名单、配对策略、群组@提及规则 |
| **出站格式化** | 处理各平台Markdown方言、消息分片、媒体上传 |

配置示例：

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+1234567890"],
      "groups": { "*": { "requireMention": true } }
    }
  }
}
```

### 2️⃣ Gateway Control Plane（网关控制平面）

位于 `src/gateway/server.ts`，是整个系统的**单一事实来源** ：

-   **网络绑定**：默认仅监听 `127.0.0.1`，远程访问需SSH隧道或Tailscale
    
-   **协议安全**：所有WebSocket帧通过JSON Schema校验（基于TypeBox生成）
    
-   **事件驱动**：客户端订阅 `agent`/`presence`/`health` 等事件，非轮询
    
-   **幂等控制**：副作用操作需携带 `idempotency key` 防止重复执行
    

### 3️⃣ Agent Runtime（Agent运行时）

位于 `src/agents/piembeddedrunner.ts`，执行AI交互的核心循环 ：

```text
1. Session Resolution → 解析消息归属的会话（main/dm:xxx/group:xxx）
2. Context Assembly   → 组装系统提示+历史+记忆搜索+技能注入
3. Model Invocation   → 流式调用配置的LLM提供商
4. Tool Execution     → 拦截工具调用，在Docker沙箱或本机执行
5. State Persistence  → 将会话状态持久化到磁盘
```

**系统提示词架构**（可组合，无需改代码）：

-   `AGENTS.md`：核心能力与约束（必需）
    
-   `SOUL.md`：人格与语气风格（可选）
    
-   `TOOLS.md`：工具使用约定（可选）
    
-   `skills/<name>/SKILL.md`：任务技能手册
    
-   动态注入：会话历史 + 语义记忆检索结果
    

### 4️⃣ Data & State Management（数据与状态）

| 存储位置 | 内容  | 安全策略 |
| --- | --- | --- |
| `~/.openclaw/openclaw.json` | 主配置（JSON5格式，支持注释） | 环境变量覆盖配置 |
| `~/.openclaw/sessions/` | 会话JSON文件（append-only事件日志） | 自动压缩旧历史防上下文溢出 |
| `~/.openclaw/memory/<agent>.sqlite` | 语义记忆索引（SQLite+向量嵌入） | 混合搜索：向量相似度+BM25关键词 |
| `~/.openclaw/credentials/` | 渠道认证凭证 | 文件权限0600，排除版本控制 |

---

## 🔐 安全架构（纵深防御）

### 多层防护体系

```text
1. 网络安全
   ├─ 默认绑定loopback，远程需SSH隧道/Tailscale Serve
   └─ Tailscale Funnel模式需密码认证

2. 认证与配对
   ├─ Token/Password认证非本地连接
   ├─ 设备身份+挑战响应握手
   └─ DM配对：未知发件人需管理员批准

3. 渠道访问控制
   ├─ allowFrom白名单
   ├─ 群组@提及策略
   └─ 配对码审批流程

4. 工具沙箱
   ├─ main会话：本机执行（高权限）
   ├─ dm/group会话：默认Docker沙箱隔离
   └─ 容器临时创建+资源限制+网络可选

5. Prompt注入防御
   ├─ 上下文隔离：用户输入≠系统指令
   ├─ 工具结果结构化封装
   └─ 推荐高能力模型+最小权限原则
```

### 关键安全设计：Lane Queue（串行队列）

> 知乎分析指出：OpenClaw架构最精妙的设计是 **Lane Queue**

-   大部分AI框架用 `async/await` 并发，OpenClaw**默认串行执行**
    
-   优势：避免竞态条件、简化状态管理、天然防重入攻击
    
-   代价：吞吐量略低，但换取了更强的可预测性和安全性
    

---

## 🔌 扩展机制（插件系统）

插件位于 `extensions/`，通过 `package.json` 的 `openclaw.extensions` 字段声明 ：

| 插件类型 | 扩展能力 | 示例  |
| --- | --- | --- |
| Channel | 新增消息渠道 | Microsoft Teams, Matrix |
| Memory | 替换存储后端 | Pinecone向量库, 知识图谱 |
| Tool | 添加自定义工具 | 企业API调用, 数据库查询 |
| Provider | 支持新模型提供商 | 本地模型, 私有化部署 |

插件热加载，无需重启Gateway。

---

## 📦 部署架构模式

| 模式  | 适用场景 | 安全建议 |
| --- | --- | --- |
| **本地开发** (macOS/Linux) | 学习/调试 | loopback绑定，无需认证 |
| **macOS菜单栏应用** | 个人日常使用 | 本地访问+Voice Wake集成 |
| **Linux VPS + SSH隧道** | 24/7远程服务 | 推荐默认方案，加密隧道 |
| **Tailscale Serve** | 多设备安全访问 | tailnet内HTTPS，自动TLS |
| **Fly.io容器部署** | 云原生托管 | 需强认证+公网防护 |

---

## 🔄 端到端消息流程（以WhatsApp为例）

```text
1️⃣ Ingestion
   WhatsApp事件 → Baileys库 → 适配器解析

2️⃣ Access Control & Routing
   白名单检查 → 配对验证 → 会话解析 (main/dm/group)

3️⃣ Context Assembly
   加载会话历史 + 组合系统提示 + 语义记忆检索

4️⃣ Model Invocation
   流式调用配置的LLM (Claude/GPT/Gemini...)

5️⃣ Tool Execution
   拦截工具调用 → Docker沙箱执行 → 结果回流模型

6️⃣ Response Delivery
   响应分块 → 适配器格式化 → 通过渠道发回用户
   ↓
   最终：会话状态持久化到磁盘
```

典型延迟预算：访问控制<10ms，会话加载<50ms，首token 200-500ms，工具执行100ms~3s

---

## 💡 核心设计哲学

1.  **Workspace-First**：Agent配置与状态以文件形式存储在工作区目录，可版本控制、可审计
    
2.  **本地优先**：对话历史、记忆、配置全部存储在用户控制的设备上
    
3.  **接口与执行分离**：消息来源（微信/电报等）与Agent运行时解耦，一处配置，多端可用
    
4.  **安全默认**：沙箱隔离、配对审批、最小权限，开箱即安全
    

---

## 📚 学习资源

-   官方文档：https://docs.openclaw.ai
    
-   中文文档：https://github.com/yeuxuan/openclaw-docs
    
-   架构深度解析：https://ppaolo.substack.com/p/openclaw-system-architecture-overview
    
-   安全审计指南：`openclaw security audit --fix` 命令
    

> OpenClaw 的本质是 **"为AI提供操作系统"** —— LLM提供智能，OpenClaw提供执行环境、会话管理、工具沙箱和安全边界 [[4]]。这种架构让个人用户能在完全掌控数据的前提下，获得强大的自动化助理能力。