# DEEP RESEARCH AI 编程代理
# Reasonix-Rust SDD 需求文档

> 日期: 2026/5/19  
> 项目: `reasonix-rust`  
> 参考项目: DeepSeek-Reasonix (TypeScript v0.44.1)  
> 仓库: [github.com/esengine/DeepSeek-Reasonix](https://github.com/esengine/DeepSeek-Reasonix)

---

## ⚡ 核心摘要

| 维度 | 内容 |
| --- | --- |
| **核心目标** | 将 DeepSeek-Reasonix 项目用 Rust 重写，并添加革命性的 `/evolve` 自我进化功能 |
| **性能与安全** | 利用 Rust 的特性，确保新版本在内存占用和执行速度上显著优于 TypeScript 原版 |
| **开发范式** | 规范驱动开发 (SDD)，使用 GitHub Spec Kit 工具链确保开发过程严谨 |
| **参考底座** | 本地项目 `~/src/DeepSeek-Reasonix` — 当前 npm 包 v0.44.1，Node ≥22，TypeScript 5.6+ |

### 参考项目现状

| 维度 | 详情 |
| --- | --- |
| **语言** | TypeScript 5.6+, ES2022, ESM (`"type": "module"`) |
| **运行时** | Node.js ≥22 |
| **CLI** | Commander.js + Ink 5 (React 18) TUI |
| **测试** | Vitest 2.x |
| **Lint** | Biome 1.9 (2-space, 双引号, 分号, 100 宽度) |
| **构建** | tsup (bundle), `tsx` (dev runner) |
| **MCP** | stdio + SSE transports，进程内 fake 用于测试 |
| **许可证** | MIT |
| **当前版本** | 0.44.1 |

### 初始化指令

```bash
# 安装 specify CLI（假设已安装 uv）
uv tool install specify-cli --from git+https://github.com/github/spec

# 初始化项目
specify init reasonix-rust --here --ai reasonix
```

---

## 1. 项目概述

本项目旨在创建一个高性能、内存安全且具备自我进化能力的 AI 编程代理。我们将严格遵循 Spec Kit 的六阶段工作流，确保每一步都有清晰、可执行的规范。

### 1.1 参考项目架构（TypeScript 原版）

原版 DeepSeek-Reasonix 采用模块化架构，核心模块如下：

```
src/
├── cli/            # CLI 入口 + commands (chat.tsx, code.tsx, diff.ts, etc.) + Ink TUI in ui/
├── tools/          # 工具定义 (filesystem, shell, MCP, plan, subagent, web, workspace)
├── mcp/            # MCP client + transports (stdio, SSE) + registry + spec
├── repair/         # 工具调用修复管道 (flatten, scavenge, storm, truncation)
├── loop.ts         # 核心事件循环 — CacheFirstLoop
├── index/          # 语义向量索引
├── code/           # SEARCH/REPLACE 编辑块解析器 + 应用门
├── core/           # 事件日志内核 (events.ts, reducers.ts, eventize.ts)
├── ports/          # 端口接口 — ModelClient, ToolHost, EventSink, MemoryStore, HookRunner, CheckpointStore
├── adapters/       # 端口具体适配器 (event-sink-jsonl.ts, event-source-jsonl.ts)
├── frame/          # Frame 编译器 (cell grid → ANSI) 用于 TUI 日志渲染
├── memory/         # 项目 / 会话 / 用户 / 运行时内存存储
├── transcript/     # Transcript 日志（写入）、diff、重放
├── telemetry/      # 使用记录 + 跨会话统计
├── server/         # Dashboard HTTP 服务器 + REST API
├── at-mentions.ts  # @提及 文件/路径 自动补全
├── client.ts       # DeepSeek API 客户端
├── config.ts       # 配置管理
├── context-manager.ts # 上下文管理器
├── hooks.ts        # 钩子系统
└── prompt-fragments.ts # 提示词片段
```

### 1.2 项目范围

- 完全重写 DeepSeek-Reasonix 的核心功能（保留所有 CLI 命令和工作模式）
- 添加 `/evolve` 自我进化功能，实现空闲时自动优化代码生成策略
- 优化缓存机制，进一步降低 Token 成本
- 重构与 GitHub API 的集成，实现更高效的搜索和学习能力
- 用 Rust 的 `reasonix-render` crate 替代 Node.js Ink TUI 渲染

### 1.3 项目原则

| 原则 | 说明 |
| --- | --- |
| **性能优先** | Rust 实现必须比 TypeScript 原版在内存占用和执行速度上有显著优势 |
| **安全性第一** | 所有外部代码执行和网络请求必须在沙箱环境中进行 |
| **可维护性** | 代码结构清晰，模块化设计，拥有完善的单元测试和集成测试 |
| **可观测性** | 提供详细的日志记录，特别是 `/evolve` 模块的学习和决策过程 |
| **兼容性** | 保持与原版 DeepSeek-Reasonix 相同的 CLI 命令和配置方式 |

---

## 2. 功能需求

### 2.1 核心功能（Core Features）

#### AI 编程代理

- 能够理解自然语言指令，生成、修改和调试代码
- 支持多种编程语言（Python, JavaScript, TypeScript, Rust 等）
- 能够读取并理解项目文件结构、现有代码和依赖关系
- 能够安全地调用外部工具（如 `git`, `cargo`, `npm`）来执行操作
- 保持原版的 `CacheFirstLoop` 设计理念：缓存优先、flash-first 成本控制、工具调用修复

#### 工作模式

| 命令 | 功能 | 说明 |
| --- | --- | --- |
| `reasonix code` | 项目级编码代理 | 支持代码搜索、替换和修改；支持 session 持久化 |
| `reasonix chat` | 纯聊天模式 | 无文件系统工具，适合技术讨论和头脑风暴 |
| `reasonix run` | 一次性任务 | 流式输出到 stdout，可直接嵌入 Shell 管道 |
| `reasonix setup` | 初始化向导 | 首次运行配置 API key |
| `reasonix doctor` | 诊断命令 | 检查系统状态 |

#### 核心命令支持

| 命令 | 功能 |
| --- | --- |
| `/help` | 查看所有命令 |
| `/pro` | 下一轮切换到 Pro 模型 (deepseek-v4-pro) |
| `/preset max` | 整个会话使用 Pro 模型 |
| `/apply` | 确认应用代码修改 |
| `/undo` | 撤回修改 |
| `/commit` | 一键 git commit |
| `/diff` | 查看差异 |
| `/skill new <名称>` | 创建技能文档 |
| `/skill list` | 列出所有技能 |
| `/skill use <名称>` | 应用特定技能 |
| `/session save` | 保存当前会话 |
| `/session load` | 加载保存的会话 |
| `/session list` | 列出所有保存的会话 |
| `/session delete` | 删除特定会话 |
| `/plan` | 开启计划模式 |
| `/new` | 开始新会话 |
| `/checkpoint` | 创建检查点 |

#### 缓存机制

- 实现高效的前缀缓存策略，确保长会话缓存命中率保持在 99.8% 以上
- 提供缓存统计功能，可查看缓存命中率、节省的 Token 数量等指标
- 支持缓存清理命令，可手动或自动管理缓存大小
- 保持原版的 `ImmutablePrefix` → `AppendOnlyLog` → `VolatileScratch` 三层缓存架构

#### 技能系统

- 支持通过 Markdown 脚本创建可复用的技能文档
- 能够自动识别技能文档并提供智能提示
- 支持技能的导入、导出和共享
- 保持原版 `.reasonix/skills/` 目录结构

#### 其他核心功能

| 功能 | 说明 |
| --- | --- |
| **MCP 协议** | 支持 stdio + SSE + Streamable HTTP transports |
| **工具系统** | ToolRegistry, 并行安全/只读标记, 计划模式门控 |
| **修复管道** | flatten → scavenge → storm → truncation (4 层工具调用修复) |
| **上下文管理** | ContextManager + thinking mode 切换 |
| **事件系统** | Event 联合类型 + reducers 纯投影 |
| **仪表盘** | Dashboard HTTP 服务器 + REST API |
| **ACL 协议** | Agent Client Protocol 支持 |
| **桌面应用** | Tauri 桌面壳 |

### 2.2 新增功能：`/evolve` 自我进化模块

#### 核心功能

- **空闲时激活**：当主任务队列为空且系统资源充足时，自动启动进化功能
- **算法研究**：
  - 向 GitHub 查找与「提升编程正确率」「AI 交互优化」「Token 节省算法」相关的优秀开源项目和论文
  - 分析这些项目的 `README.md`、核心算法文档和代码实现
  - 从搜索结果中提取有效的 Prompt Engineering 模式、代码生成策略或推理优化技巧
- **策略学习与集成**：
  - 将学到的策略安全地集成到自身的提示词模板或推理逻辑中
  - 所有变更必须记录在案，并可通过版本控制回滚
  - 提供进化历史查看命令，记录每次学习的来源和效果
- **效果验证**：
  - 在内部测试集上运行，对比进化前后的代码生成质量
  - 仅当指标显著提升时，才将新策略设为默认

#### 自我进化流程

```
1. 空闲检测 → 2. 算法研究 → 3. 策略集成 → 4. 效果验证
     ↑                                          |
     └────────── 持续循环优化 ←─────────────────┘
```

1. **空闲检测** — 当主任务队列为空且系统资源充足时，自动启动进化功能
2. **算法研究** — 在 GitHub 上搜索相关优秀项目，分析其核心算法和文档
3. **策略集成** — 将学到的策略安全地集成到自身的提示词模板或推理逻辑中
4. **效果验证** — 在内部测试集上运行，对比进化前后的代码生成质量，决定是否应用新策略

#### 进阶功能

- 支持手动触发 `/evolve`，可指定搜索关键词或范围
- 提供进化进度查看命令，显示当前的学习状态
- 支持进化过程的暂停和继续
- 提供进化结果的详细报告，包括学习的算法、预期效果和实际性能提升

### 2.3 非功能性需求

#### 性能指标

| 指标 | 目标 |
| --- | --- |
| 内存占用 | 比 TypeScript 原版降低 50% 以上 |
| 响应时间 | 比 TypeScript 原版快 30% 以上 |
| Token 消耗 | 比 TypeScript 原版降低 20% 以上 |
| 并发能力 | 支持至少 100 个并发请求 |
| 缓存命中率 | 保持 99.8% 以上 |

#### 安全要求

- 所有外部代码执行必须在沙箱环境中进行
- 文件系统访问必须受到严格限制，仅允许访问项目目录
- 网络请求必须通过代理服务器进行，避免直接暴露 IP
- 敏感信息（如 GitHub Personal Access Token、DeepSeek API Key）必须安全存储
- 使用 Rust 的内存安全特性避免 TypeScript 原版中可能存在的不安全模式

#### 可维护性

- 代码必须符合 Rust 的 idiomatic 编程规范
- 模块化设计，核心功能与辅助功能分离
- 提供完善的文档，包括 API 文档和用户指南
- 每个模块必须有单元测试覆盖率超过 80%

#### 可观测性

- 提供实时仪表盘，显示 Token 消耗、缓存命中率和成本
- 支持详细的日志记录，包括请求、响应和内部决策过程
- 提供性能监控功能，可查看系统资源使用情况
- 支持错误追踪和报告，便于快速定位和修复问题
- 特别是在 `/evolve` 模块中：记录每次学习的搜索关键词、来源仓库、提取策略和效果评估

---

## 3. 技术规划

### 3.1 技术栈选择

| 层面 | 选择 | 说明 |
| --- | --- | --- |
| **核心语言** | Rust (稳定版 1.70+) | 内存安全 + 高性能，适合处理大量网络请求和文件操作 |
| **异步运行时** | `tokio` 1.0+ | Rust 生态最成熟的异步运行时 |
| **HTTP 客户端** | `reqwest` + `octocrab` | 通用 HTTP + GitHub API 类型安全 |
| **序列化** | `serde` + `ron` | JSON/YAML 序列化 + 人类可读配置 |
| **内存缓存** | `DashMap` | 高性能线程安全并发缓存 |
| **持久化存储** | `SQLite` (via `rusqlite`) | 轻量级持久化：配置、技能文档、进化历史 |
| **日志** | `tracing` + `env_logger` | 异步追踪 + 简单配置 |
| **CLI** | `clap` 4.x | Rust CLI 标准库，替换 Commander.js |
| **TUI** | `ratatui` | Rust TUI 框架 |
| **测试** | `cargo test` + `cargo-nextest` + `tokio::test` | 标准 + 并行 + 异步测试 |
| **基准测试** | `criterion` | 性能基准测试 |

### 3.2 架构设计

#### 模块架构

```
┌─────────────────────────────────────────────────┐
│                      CLI (clap)                  │
├─────────────────────────────────────────────────┤
│  core          │  cache      │  github          │
│  (代理逻辑)    │  (前缀缓存) │  (API 集成)      │
├─────────────────────────────────────────────────┤
│  skills        │  evolve     │  mcp             │
│  (技能系统)    │  (自我进化) │  (MCP 协议)     │
├─────────────────────────────────────────────────┤
│  repair        │  sandbox    │  server          │
│  (工具修复)    │  (安全执行) │  (仪表盘)        │
├─────────────────────────────────────────────────┤
│                     utils                        │
│  (配置、日志、telemetry、context-manager)        │
└─────────────────────────────────────────────────┘
```

#### 核心组件

| 组件 | 职责 | 对应原版 |
| --- | --- | --- |
| **模型调用层** | 与 DeepSeek API 交互，提示词生成和响应解析 | `src/client.ts` |
| **缓存层** | 前缀缓存策略，减少重复请求 | `src/memory/runtime.ts` (ImmutablePrefix, AppendOnlyLog, VolatileScratch) |
| **沙箱层** | 安全代码执行环境，限制文件系统和网络访问 | 新增 |
| **学习层** | `/evolve` 核心逻辑：搜索、分析、集成 | 新增 |
| **修复层** | 工具调用修复管道 | `src/repair/` |
| **MCP 层** | MCP 协议客户端和服务注册 | `src/mcp/` |
| **仪表盘** | HTTP 服务器 + Web UI | `src/server/` + `dashboard/` |

#### 组件交互

```
CLI (clap)
    ↓
Core (CacheFirstLoop)
    ←→ Cache (DashMap + SQLite)
    ←→ Sandbox (安全执行)
    → GitHub API (octocrab)
    → Evolve (学习引擎)
    → Skills (技能管理)
    → MCP (外部工具)
```

### 3.3 TypeScript → Rust 迁移映射

| TypeScript 原版 | Rust 目标 | 说明 |
| --- | --- | --- |
| `src/cli/index.ts` (Commander.js) | `src/cli/` (clap) | CLI 入口和命令注册 |
| `src/cli/commands/code.tsx` (Ink TUI) | `src/cli/commands/code.rs` (ratatui) | code 命令 TUI |
| `src/cli/commands/chat.tsx` | `src/cli/commands/chat.rs` | chat 命令 TUI |
| `src/loop.ts` (CacheFirstLoop) | `src/core/loop.rs` | 核心事件循环 |
| `src/client.ts` (DeepSeekClient) | `src/core/client.rs` | API 客户端 |
| `src/memory/runtime.ts` | `src/cache/runtime.rs` | 三层缓存架构 |
| `src/tools/` (ToolRegistry) | `src/tools/` | 工具注册与调度 |
| `src/repair/` (修复管道) | `src/repair/` | 工具调用修复 |
| `src/mcp/` (MCP 协议) | `src/mcp/` | MCP 客户端+传输 |
| `src/server/` (HTTP 仪表盘) | `src/server/` | 仪表盘服务器 |
| `src/code/edit-blocks.ts` | `src/code/edit_blocks.rs` | SEARCH/REPLACE 编辑 |
| `src/hooks.ts` | `src/hooks.rs` | 钩子系统 |
| `src/context-manager.ts` | `src/context/manager.rs` | 上下文管理 |
| `src/at-mentions.ts` | `src/at_mentions.rs` | @提及补全 |
| `src/prompt-fragments.ts` | `src/prompts/fragments.rs` | 提示词片段 |
| `crates/reasonix-render/` | 保留并扩展 | ANSI 渲染 crate |

---

## 4. 任务拆解

### 4.1 核心迁移任务

#### 缓存机制重写

- 重构缓存系统，使用 `DashMap` 替代 TypeScript 原版的 `ImmutablePrefix` / `AppendOnlyLog` / `VolatileScratch`
- 优化前缀缓存策略，确保高命中率（99.8%+）和低 Token 消耗
- 实现缓存统计功能，可查看缓存命中率、节省的 Token 数量等指标
- 添加缓存清理命令，支持手动和自动管理缓存大小

#### 模型调用层重构

- 重构模型调用接口，支持多种 AI 模型的灵活切换（DeepSeek V3, V4, R1 等）
- 优化提示词生成逻辑，减少冗余信息和 Token 消耗
- 实现模型切换命令（`/pro`、`/preset`）的底层逻辑
- 添加模型性能监控，可查看不同模型的响应时间和 Token 消耗
- 实现 escalation 机制（flash → pro 自动切换）

#### 代码执行沙箱

- 实现安全的代码执行环境，限制文件系统和网络访问
- 重构技能系统，支持通过 Markdown 脚本创建可复用的技能文档
- 实现技能的导入、导出和共享功能
- 添加技能应用命令（`/skill use <名称>`）的底层逻辑

#### 命令行接口

- 重构 CLI 模块，使用 `clap` 保持与原版相同的命令和配置方式
- 优化命令解析逻辑，提高响应速度和准确性
- 实现命令历史记录和自动补全功能
- 添加 CLI 的性能监控和日志记录

### 4.2 新增功能：`/evolve` 自我进化模块

#### 空闲检测与调度

- 实现空闲检测逻辑，判断主任务队列是否为空
- 设计资源充足检测机制，确保系统资源足够支持进化过程
- 实现异步任务调度，确保进化过程不影响主功能
- 添加进化触发配置，允许用户自定义触发条件和频率

#### GitHub API 集成

- 实现 GitHub 搜索功能，查找与目标相关的仓库
- 设计搜索参数优化策略，提高搜索结果的相关性
- 实现分页请求逻辑，处理 GitHub API 的分页限制
- 添加速率限制处理，避免触发 GitHub 的 API 限制

#### 算法学习与集成

- 设计算法提取规则，从仓库中提取有效的策略和模式
- 实现策略验证机制，确保集成的策略不会破坏系统稳定性
- 设计版本控制集成，自动记录和回滚策略变更
- 实现进化历史记录功能，可查看每次学习的来源和效果

#### 效果验证与反馈

- 设计内部测试集，用于验证进化后的代码生成质量
- 实现性能指标收集和分析，对比进化前后的各项指标
- 设计自动反馈机制，根据测试结果决定是否应用新策略
- 添加进化结果报告功能，提供详细的性能提升分析

### 4.3 测试与优化任务

#### 单元测试

- 为每个模块编写单元测试，覆盖核心功能和边界条件
- 使用 `tokio::test` 编写异步测试，验证异步逻辑的正确性
- 实现测试覆盖率统计，确保每个模块的测试覆盖率超过 80%
- 添加测试性能分析，优化测试执行时间和资源占用

#### 集成测试

- 编写 CLI 模块的集成测试，验证命令和配置的正确性
- 实现缓存系统的集成测试，验证高命中率和低 Token 消耗
- 编写模型调用层的集成测试，验证不同模型的兼容性和性能
- 添加代码执行沙箱的集成测试，验证安全性和限制机制

#### 性能测试

- 使用 `criterion` 测量核心功能的性能指标
- 比较 Rust 版与 TypeScript 原版的内存占用和执行速度
- 测试 Token 消耗情况，验证缓存优化的效果
- 评估高并发场景下的系统稳定性

#### 安全测试

- 使用 `tempdir` 创建临时目录，测试沙箱对文件系统的限制
- 设计安全漏洞测试场景，验证系统的安全性
- 测试网络请求的代理配置，确保敏感信息不被暴露
- 验证 DeepSeek API Key 和 GitHub Token 的安全存储机制

---

## 5. 实现步骤

### 5.1 缓存机制重写

**步骤：**
1. 使用 `DashMap` 实现线程安全的内存缓存
2. 设计高效的前缀缓存策略，优化缓存键的生成和存储
3. 实现缓存统计功能，记录命中率、节省的 Token 数量等指标
4. 添加缓存清理命令，支持手动和自动管理缓存大小

**预期成果：**
- 缓存命中率保持在 99.8% 以上
- Token 消耗比 TypeScript 原版降低 20% 以上
- 缓存查询速度提高 30% 以上
- 提供详细的缓存统计报告

### 5.2 模型调用层重构

**步骤：**
1. 设计通用的模型调用接口，支持多种 AI 模型的灵活切换
2. 优化提示词生成逻辑，减少冗余信息和 Token 消耗
3. 实现模型切换命令（`/pro`、`/preset`）的底层逻辑，确保命令正确执行
4. 添加模型性能监控，收集和分析不同模型的响应时间和 Token 消耗

**预期成果：**
- 支持至少 5 种主流 AI 模型（DeepSeek V3, V4, R1, GPT, Claude 等）
- 模型切换命令响应时间低于 200ms
- 提供模型性能比较报告，帮助用户选择最佳模型
- 优化提示词生成，减少 20% 的 Token 消耗

### 5.3 代码执行沙箱

**步骤：**
1. 设计安全的代码执行环境，限制文件系统和网络访问
2. 实现技能系统的底层逻辑，支持 Markdown 脚本的解析和应用
3. 添加技能导入、导出和共享功能，支持技能文档的版本控制
4. 优化 CLI 的交互体验，确保命令历史记录和自动补全功能的稳定性

**预期成果：**
- 代码执行环境完全隔离，无法访问项目目录外的文件
- 技能文档的解析和应用速度提高 50% 以上
- 支持技能文档的导入、导出和共享
- CLI 的命令历史记录和自动补全功能稳定可靠

### 5.4 `/evolve` 自我进化模块实现

**步骤：**
1. 实现空闲检测逻辑，判断主任务队列是否为空
2. 设计 GitHub API 搜索策略，精准查找相关仓库
3. 实现分页请求逻辑，处理 GitHub API 的分页限制
4. 添加速率限制处理，确保不会触发 GitHub 的 API 限制

**预期成果：**
- 空闲检测准确率超过 95%
- GitHub API 搜索结果的相关性超过 80%
- 分页请求逻辑高效稳定，支持大规模数据检索
- 速率限制处理机制有效，避免触发 GitHub 的 API 限制

### 5.5 整合与优化

**步骤：**
1. 整合所有模块，确保系统整体功能稳定可靠
2. 优化系统性能，确保内存占用比 TypeScript 原版降低 50% 以上
3. 测试系统在高并发场景下的稳定性
4. 优化 CLI 的交互体验，提高响应速度和准确性

**预期成果：**
- 系统整体功能稳定可靠，通过所有测试用例
- 内存占用比 TypeScript 原版降低 50% 以上
- 响应时间比 TypeScript 原版快 30% 以上
- CLI 的交互体验流畅，响应时间低于 500ms

---

## 6. 审查与测试

### 6.1 测试策略

| 层级 | 工具 | 目标 |
| --- | --- | --- |
| **单元测试** | `tokio::test` + `cargo test` | 每模块覆盖率 >80% |
| **集成测试** | `cargo test --test '*'` | CLI / 缓存 / 模型调用 / 沙箱 |
| **性能测试** | `criterion` | 内存 / 速度 / Token 对比 |
| **安全测试** | `tempdir` + 手动渗透 | 沙箱隔离 / 网络代理 / Token 安全 |

### 6.2 测试用例

#### 缓存机制测试

- 测试缓存键的生成和存储逻辑，验证前缀缓存策略的正确性
- 测试缓存命中率统计功能，验证统计指标的准确性
- 测试缓存清理命令，验证手动和自动清理机制的效果
- 测试大规模缓存操作，验证系统的稳定性和性能

#### 模型调用层测试

- 测试不同模型的切换逻辑，验证命令的正确执行
- 测试提示词生成逻辑，验证 Token 消耗的减少效果
- 测试模型性能监控，验证数据收集和分析的准确性
- 测试多模型并发请求，验证系统的稳定性和响应速度

#### 代码执行沙箱测试

- 测试文件系统访问限制，验证沙箱的安全性
- 测试网络请求的代理配置，验证网络访问的限制
- 测试技能系统的解析和应用逻辑，验证技能的正确性
- 测试 CLI 的命令历史记录和自动补全功能

#### 自我进化模块测试

- 测试空闲检测逻辑，验证检测的准确性和及时性
- 测试 GitHub API 搜索策略，验证搜索结果的相关性和质量
- 测试分页请求逻辑，验证处理大规模数据的能力
- 测试速率限制处理，验证系统的稳定性和可靠性

---

## 7. 宪法文件（Constitution）

### 7.1 代码质量标准

- 遵循 Rust 的 idiomatic 编程规范（`rustfmt` + `clippy`）
- 每个函数和模块必须有清晰的文档注释（`///`）
- 代码必须通过 `clippy` 的严格检查
- 提供完善的错误处理和恢复机制（`anyhow` / `thiserror`）

### 7.2 测试规范

- 每个模块必须有单元测试，覆盖率超过 80%
- 核心功能必须有集成测试，验证功能的正确性和稳定性
- 性能关键路径必须有基准测试（`criterion`），确保性能指标达标
- 安全相关功能必须有专门的安全测试，验证系统的安全性

### 7.3 用户体验一致性要求

- 保持与原版 DeepSeek-Reasonix 相同的 CLI 命令和配置方式
- 优化交互体验，提高命令的响应速度和准确性
- 提供清晰的错误提示和帮助文档
- 设计友好的日志和仪表盘，提供详细的系统状态信息

### 7.4 性能要求

| 指标 | 目标 | 对比基准 |
| --- | --- | --- |
| 内存占用 | 降低 50%+ | TypeScript 原版 v0.44.1 |
| 响应时间 | 快 30%+ | TypeScript 原版 v0.44.1 |
| Token 消耗 | 降低 20%+ | TypeScript 原版 v0.44.1 |
| 并发请求 | 100+ | 稳定运行 |

### 7.5 安全要求

- 所有外部代码执行必须在沙箱环境中进行
- 文件系统访问必须受到严格限制，仅允许访问项目目录
- 网络请求必须通过代理服务器进行，避免直接暴露 IP
- 敏感信息（DeepSeek API Key、GitHub PAT）必须安全存储

### 7.6 可观测性要求

- 提供详细的日志记录，包括请求、响应和内部决策过程
- 实现性能监控功能，可查看系统资源使用情况
- 设计实时仪表盘，显示 Token 消耗、缓存命中率和成本
- 支持错误追踪和报告，便于快速定位和修复问题

---

## 8. 实现与审查

### 8.1 实现步骤

#### 缓存机制实现

1. 使用 `DashMap` 创建线程安全的缓存结构
2. 实现高效的前缀缓存策略，优化缓存键的生成和存储
3. 编写缓存统计功能，记录命中率、节省的 Token 数量等指标
4. 添加缓存清理命令，支持手动和自动管理缓存大小

#### 模型调用层实现

1. 设计通用的模型调用接口，支持多种 AI 模型的灵活切换
2. 优化提示词生成逻辑，减少冗余信息和 Token 消耗
3. 实现模型切换命令的底层逻辑，确保命令正确执行
4. 添加模型性能监控，收集和分析不同模型的响应时间和 Token 消耗

#### 代码执行沙箱实现

1. 设计安全的代码执行环境，限制文件系统和网络访问
2. 实现技能系统的底层逻辑，支持 Markdown 脚本的解析和应用
3. 添加技能导入、导出和共享功能，支持技能文档的版本控制
4. 优化 CLI 的交互体验，确保命令历史记录和自动补全功能的稳定性

#### 自我进化模块实现

1. 实现空闲检测逻辑，判断主任务队列是否为空
2. 设计 GitHub API 搜索策略，精准查找相关仓库
3. 实现分页请求逻辑，处理 GitHub API 的分页限制
4. 添加速率限制处理，确保不会触发 GitHub 的 API 限制

### 8.2 审查与测试

#### 代码审查

```bash
# 代码质量
cargo clippy -- -D warnings

# 正确性
cargo check

# 格式化
cargo fmt -- --check

# 单元测试
cargo test

# 并行测试
cargo nextest run
```

#### 性能审查

```bash
# 基准测试
cargo bench

# 比较 Rust vs TypeScript 原版
# 内存占用、执行速度、Token 消耗
```

#### 安全审查

- 使用 `tempdir` 创建临时目录，测试沙箱对文件系统的限制
- 设计安全漏洞测试场景，验证系统的安全性
- 测试网络请求的代理配置，确保敏感信息不被暴露
- 验证 DeepSeek API Key 和 GitHub Token 的安全存储机制

#### 用户体验审查

```bash
# 运行 CLI 测试所有命令
cargo run -- code
cargo run -- chat
cargo run -- run "echo hello"

# 比较 Rust vs TypeScript 原版交互体验
```

---

## 9. 部署与集成验证

### 9.1 部署策略

```bash
# 安装可执行文件
cargo install --path .

# 发布到 crates.io
cargo publish

# Docker 镜像
docker build -t reasonix-rust .
docker run -it --rm reasonix-rust code
```

- 提供详细的安装和配置指南
- 支持通过 `cargo publish` 发布到 Rust 包仓库
- 提供 Docker 镜像，便于容器化部署
- 提供各平台预编译二进制（GitHub Releases）

### 9.2 集成验证

- 测试 CLI 模块与核心功能的集成
- 验证缓存机制与模型调用层的协同工作
- 测试代码执行沙箱的安全性和限制机制
- 验证自我进化模块的学习和集成能力

### 9.3 性能验证

- 比较 Rust 版与 TypeScript 原版的内存占用和执行速度（使用原版 benchmark 套件）
- 测试 Token 消耗情况，验证缓存优化的效果
- 评估高并发场景下的系统稳定性
- 测试长期运行的系统稳定性

### 9.4 安全验证

- 测试文件系统访问限制，验证沙箱的安全性
- 测试网络请求的代理配置，确保敏感信息不被暴露
- 验证 DeepSeek API Key 和 GitHub Token 的安全存储机制
- 测试自我进化模块的学习和集成过程，确保不会引入安全漏洞

### 9.5 用户体验验证

- 测试所有 CLI 命令和配置的正确性（与 TypeScript 原版一一对照）
- 验证缓存机制对 Token 消耗的减少效果
- 测试技能系统的解析和应用逻辑，验证技能的正确性
- 验证自我进化模块的学习和集成能力，确保系统能够持续优化

---

## 10. 总结与展望

### 项目总结

- 本项目将 DeepSeek-Reasonix 用 Rust 语言重新实现，提升性能和安全性
- 新增 `/evolve` 自我进化功能，实现空闲时自动优化代码生成策略
- 重构缓存机制，进一步降低 Token 消耗
- 优化与 GitHub API 的集成，实现更高效的搜索和学习能力

### 技术亮点

| 亮点 | 说明 |
| --- | --- |
| **DashMap 缓存** | 高性能、线程安全的缓存系统，替代 TypeScript 原版的三层缓存 |
| **tokio 异步** | 利用 Rust 的零成本抽象优化系统性能 |
| **安全沙箱** | 设计安全的代码执行沙箱，确保系统的安全性 |
| **自我进化** | 让系统能够持续学习和优化 |
| **零成本 FFI** | 保留 `reasonix-render` crate，与 Rust CLI 原生集成 |

### 未来展望

```
当前版本                  未来版本                    远景目标
Rust 重写 & Self-Evolution → 扩展语言/模型支持 & 高级功能 → AI 工具链集成 & 生态构建
```

- 扩展支持更多编程语言和 AI 模型
- 优化自我进化模块的学习算法，提高学习效率
- 添加更多高级功能，如代码质量分析、自动测试生成等
- 探索与其他 AI 工具的集成，提供更丰富的开发体验

### 项目意义

- 为 AI 编程代理领域提供一个高性能、内存安全的参考实现
- 探索规范驱动开发 (SDD) 在 AI 工具开发中的应用
- 验证自我进化功能在编程代理中的可行性
- 为开源社区贡献一个高质量的 Rust 项目

---

## 附录 A：Spec Kit 工作流

```bash
# 1. 初始化
specify init reasonix-rust --here --ai reasonix

# 2. 宪法审查
specify review --performance
specify review --security
specify review --ux

# 3. 生成宪法文件
# → specifications/constitution.md

# 4. 开发
specify dev

# 5. 测试
cargo test
cargo nextest run

# 6. 构建
cargo build --release

# 7. 部署
cargo install --path .

# 8. 启用 /evolve
reasonix code
# 等待空闲... 观察自动进化过程
```

## 附录 B：TypeScript 原版参考文件索引

| 原版文件 | 功能 |
| --- | --- |
| `src/cli/index.ts` | CLI 入口 + Commander.js 命令注册 |
| `src/cli/commands/code.tsx` | `reasonix code` 命令实现 |
| `src/cli/commands/chat.tsx` | `reasonix chat` 命令实现 |
| `src/cli/commands/run.ts` | `reasonix run` 命令实现 |
| `src/loop.ts` | CacheFirstLoop 核心事件循环 |
| `src/client.ts` | DeepSeek API 客户端 |
| `src/memory/runtime.ts` | 三层缓存 (ImmutablePrefix, AppendOnlyLog, VolatileScratch) |
| `src/tools.ts` | ToolRegistry 工具注册中心 |
| `src/tools/filesystem.ts` | 文件系统工具 |
| `src/tools/shell.ts` | Shell 执行工具 |
| `src/tools/web.ts` | Web 工具 |
| `src/tools/plan.ts` | 计划工具 |
| `src/tools/subagent.ts` | 子代理工具 |
| `src/mcp/client.ts` | MCP 客户端 |
| `src/repair/flatten.ts` | 工具调用修复：flatten |
| `src/repair/scavenge.ts` | 工具调用修复：scavenge |
| `src/repair/storm.ts` | 工具调用修复：storm |
| `src/repair/truncation.ts` | 工具调用修复：truncation |
| `src/server/index.ts` | Dashboard HTTP 服务器 |
| `src/code/edit-blocks.ts` | SEARCH/REPLACE 编辑块解析器 |
| `src/hooks.ts` | 钩子系统 |
| `src/context-manager.ts` | 上下文管理器 |
| `src/config.ts` | 配置管理 |
| `crates/reasonix-render/` | Rust ANSI 渲染 crate |

---

> 通过这份规格文档，我们明确了 `reasonix-rust` 项目的核心目标、功能需求和技术规划，为后续的开发工作提供了清晰的指导。  
> 参考项目 `DeepSeek-Reasonix` (v0.44.1) 的完整源代码位于 `~/src/DeepSeek-Reasonix`，作为 Rust 重写的权威参考底座。

> **核心信念**：通过 Rust 的性能与 SDD 的严谨，结合 `/evolve` 的自我优化能力，我们将重新定义 AI 编程代理的可能性。
