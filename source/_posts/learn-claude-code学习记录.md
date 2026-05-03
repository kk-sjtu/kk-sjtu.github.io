---
title: Learn Claude Code 学习记录
date: 2026-05-03 17:27:20
tags:
  - Claude Code
  - AI
  - 学习笔记
categories:
  - 技术
---

## 概述

本文是学习 Claude Code (LCC) 源码的学习记录，记录了 Agent 系统设计中的核心概念和实现思路。

---

## Chapter 1 - Agent Loop

基本的 `agent_loop` 循环机制，与 claw0 设计一致。

**核心思考：**
> 如果让 Agent 生成一个文件，在这个过程中如何处理 context 和接下来的 message 信息来节省 token，这是一个很重要的优化方向。

---

## Chapter 2 - Tool Use

工具调用的安全性和分发机制：

- **安全路径**：加强工具调用时的安全性
- **分发映射表**：使用 `dispatch map` 存储 tool
- **字典拆解**：使用 `**` 小技巧进行字典拆解，处理 Agent 返回的内容

---

## Chapter 3 - Todo System

通过 schema 工具表添加 todo 工具，设置相应字段，保证 Agent 对话时行动不偏移。

**工作流程：**
1. 在 N 次工具调用后触发 todo 功能
2. 将结果作为 result 传给下一轮对话
3. 下一轮对话时 Agent 返回带有 `tool: todo` 的消息
4. 提取 todo 工具字段，传入到 `TodoManager` 实例中
5. 多轮调用时，前期记录保存在 TodoManager 的属性字段中

---

## Chapter 4 - Subagent

Agent 根据用户提问、tool-list、系统提示词，判断是否需要开启 subagent。

**关键设计：**
- Subagent 结果仅将最后一条消息返回主 Agent，忽略中间过程（节省 token）
- Subagent 实现了会话隔离，防止主对话被污染
- 本章的 subagent 是**主线程阻塞**的，没有后台线程
- claw0 中使用 `asyncio` 多线程实现了后台调用子 agent
- 后续 LCC 可能也会加入多线程操作

---

## Chapter 5 - Skill Loading

**核心思想：渐进式披露 / 按需披露**

系统提示词相当于全局变量，包含 skill 目录与简介。在 Agent 的每一轮对话中：
1. 分析是否需要使用某种 skill
2. 确认使用后，读取对应的 skill 文件加入上下文
3. 读取 skill 被建模为一个工具调用

> 写好 skill 本身也很重要。

---

## Chapter 6 - Context Compact

三层压缩机制，通过压缩上下文节省 token：

### 第一层：Micro Compact
每次调用 LLM 前，将旧的 `tool-result` 替换为占位符。

**策略：战略性丢失**
- 丢弃旧的、长的内容
- 代码读取的文件内容不丢失（丢失会让 Agent 变笨，还会逼迫重读文件）
- 直接修改 message 中的内容

### 第二层：Auto Compact
上下文超出 token 阈值时，使用大模型进行摘要压缩。

**实现方式：**
- 在自动压缩函数中新调用一个 `max_token` 较小的 LLM
- 将结果打包为 summary，返回到主线程供主 Agent 使用
- 原始消息脚本进行持久化保留

### 第三层：Manual Compact
将压缩变为工具，由 LLM 按需触发。

> 信息没有真正丢失，只是移出了活跃上下文。

---

## Chapter 7 - Task System

本节添加了完整的任务管理系统，支持：创建任务、更新任务、删除依赖、显示任务。

**工作原理：**
1. 将任务管理的 CRUD 函数注入 schema
2. Schema 作为系统提示词注入 LLM 输入参数
3. LLM 根据提问选择合适的工具
4. 涉及任务管理时调用该系统

**与 Todo 的区别：**
- Todo 是内存式的，注入上下文
- Task System 是持久化的，存储在磁盘上

---

## Chapter 8 - Background Tasks

后台任务管理系统，通过工具调用启动后台任务。

**核心机制：**
- 后台任务以子进程方式启动
- `agent_loop` 每次对话前，先排空一遍消息队列
- 使用 `with self._lock` 自动加锁解锁
- 子进程执行期间，排空操作会等待
- 执行完成后解锁，进行排空

**返回格式示例：**
```json
{
  "task_id": task_id,
  "status": status,
  "command": command[:80],
  "result": (output or "(no output)")[:500]
}
```

**设计细节：**
- 排空时不返回 `command` 信息（减少上下文长度）
- `command` 信息可通过 check 函数获取

---

## Chapter 9 - Agent Teams

类似 sub-agent 但子 Agent 拥有生命周期，通过 `config.json` 维护。

### Agent 收件箱机制
每个 Agent 拥有自己的收件箱记录请求：
1. 执行 `agent_loop` 调用 LLM 前，读取 inbox 内容放入上下文
2. 清空 inbox
3. 调用 `send_message` 工具给其他 Agent 发消息（原子级写入 JSON）

```python
with open(inbox_path, "a") as f:
    f.write(json.dumps(msg) + "\n")
```

> 操作系统文件读写天然满足原子性，所以 `BUS.read_inbox` 不需要加锁。

---

## Chapter 10 - Team Protocols

在命令-执行基础上，增添更充分的协议机制。

### 案例 1：关机协议
- Lead 要求关机时，先在 shutdown 队列添加 request
- Subagent 拥有 `shutdown-response` 工具，可根据情况决定同意或拒绝
- 结果写入 shutdown 队列

### 案例 2：计划审批
- Subagent 向 Lead 提出 request（数据流动方向与关机相反）
- 同样采用审核机制

> 线程锁的目的：防止 lead 和 subagent 同时读写共享数据。

---

## Chapter 11 - Autonomous Agents

给 Agent 团队增添自主性，让 Lead Agent 自主分配任务。

### 指定任务分配
用户明确提到具体 Agent 时：
1. Lead Agent spawn 对应 Agent 的线程
2. 发送邮件到其 inbox
3. Agent 启动后读取 inbox，注入上下文并清空
4. 可能回复 shutdown-request
5. 根据 inbox 中的命令开始工作

### 自主任务分配
用户未指定具体 Agent 时：
1. 轮询 teammate 状态，跳过正在工作的
2. 找到空闲 Agent 分配
3. 全忙时自主创建新 Agent

### 空闲状态判断
Agent 读取 `config.json`，识别 "idle" 字眼后选择 IDLE 工具
- 下一个对话的 `block.name` 为 `idle`
- 只有进入空闲态时，sub-agent 才会查看任务看板

---

## Chapter 12 - Worktree + Task Isolation

虽然文件写入是原子级的，但多 Agent 共享目录会导致无法回滚问题。

> "干净回滚"指的是能够可靠地撤销某个 Agent 做的所有改动，而不影响其他 Agent 的工作。

### Worktree vs Git Branch
| 特性 | Worktree | Git Branch |
|------|----------|------------|
| 目录结构 | 物理复制多份目录 | 同一目录切换分支 |
| 适用场景 | 多 Agent 并行操作 | 单线程任务 |

### 工作流程
1. 用户提出需求后，主 Agent 创建任务
2. 根据要求创建不同 worktree，每个 worktree 绑定一项任务
3. `task.json` 保存所属 worktree 和任务 status
4. 每个 worktree 开启后台线程执行子任务（bash 命令）
5. 线程操作前后 emit event 记录到 `event.json`（持久化历史）
6. 执行完后 Agent 自主选择 `remove` 整个目录或 `keep`

> 本章没有使用 subagent，用子线程执行 bash 命令替代。
