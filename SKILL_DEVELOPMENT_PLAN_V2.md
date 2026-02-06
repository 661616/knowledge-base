# `/life-release` Skill 开发方案 v2.0
**基于 CoS (Chain of Skills) 架构的改进方案**

---

## 📊 核心架构对比

### 原方案 vs CoS 架构

| 维度 | 原方案 | CoS 架构 | 改进点 |
|------|--------|----------|--------|
| **文件组织** | 单一 Skill | 多层次上下文管理 | ✅ 更清晰的职责分离 |
| **上下文管理** | 一次性加载 | Rolling Context | ✅ 渐进式披露，节省 token |
| **工作流** | 线性流程 | 多命令组合 | ✅ 灵活性更高 |
| **可扩展性** | 单体 Skill | 模块化 | ✅ 易于迭代 |

---

## 🏗️ CoS 架构深度解析

### 1. 文件结构（参考你的图）

```
~/.claude/skills/cos/
├── SKILL.md                    # Skill 定义 + 命令说明
├── references/
│   ├── cos_instruction.md      # CoS 完整角色定义
│   └── changelog_template.md   # 周报模板
├── context/
│   ├── rolling_context.md      # 当前周累积记录（动态更新）
│   ├── changelog.md            # 周报（release notes）
│   ├── learning_input.md       # 学习 input 笔记
│   └── daily/                  # 每日 commit log
│       ├── 2026-01-19.md
│       ├── ...
│       └── 2026-01-27.md
└── rss-digest/                 # RSS 每日摘要（可选）
    ├── 2026-01-24.md
    └── 2026-01-25.md
```

### 2. 核心概念映射

| 人生概念 | 软件工程隐喻 | 对应文件 |
|---------|-------------|---------|
| **当前状态** | Runtime State | `rolling_context.md` |
| **每日记录** | Commit Log | `daily/YYYY-MM-DD.md` |
| **周报** | Release Notes | `changelog.md` |
| **版本号** | Semantic Versioning | `v1.X (周数)` |
| **不记录的事** | `.gitignore` | （概念层面） |

### 3. 工作流架构

```
日常输入 ──→ 记录 ──→ 累积 ──→ 周报 ──→ 存档
   │        │       │       │       │
   ↓        ↓       ↓       ↓       ↓
对话/事件  daily/  rolling  changelog  archive/
          log    context   (weekly)
```

---

## 🔄 Progressive Disclosure（渐进式披露）

### 理论基础

根据 Claude-Mem 和 SimpleMem 论文的研究：

**传统 RAG 问题**：一次性加载所有上下文 → 污染注意力预算

**Progressive Disclosure 方案**：
1. **Layer 1 (Index)**：只显示元数据（~1000 tokens）
2. **Layer 2 (Details)**：按需获取完整内容
3. **Layer 3 (Deep Dive)**：必要时读取原始文件

### 在 CoS 中的应用

#### **Level 1: Metadata（总是加载）**
```yaml
---
name: cos
description: 生命周期管理系统，包含 4 个子命令：/cos log, /cos review, /cos status, /cos weekly
---
```
- **Token 消耗**：~100 tokens
- **作用**：让 Claude 知道有这个 Skill

#### **Level 2: Skill Body（触发时加载）**
```markdown
# CoS (Chain of Skills) - 生命周期管理

## 作用
管理个人成长的完整生命周期，从日常记录到周报生成。

## 命令

### /cos log
日常记录，追加到 rolling_context.md
...
```
- **Token 消耗**：~2000 tokens
- **作用**：提供完整的命令说明

#### **Level 3: Bundled Resources（引用时加载）**
```markdown
参考 `references/cos_instruction.md` 了解完整角色定义
参考 `references/changelog_template.md` 了解周报模板
读取 `context/rolling_context.md` 获取当前累积内容
```
- **Token 消耗**：按需（0-10000 tokens）
- **作用**：只在需要时加载大量上下文

---

## 📝 四个核心命令设计

### 1. `/cos log` - 日常记录

**作用**：快速记录今天的活动，追加到 `rolling_context.md`

**输入**：
```
/cos log

今天完成了：
1. 学习 MCP 协议
2. 搭建 Hugo 博客
3. 运动30分钟
```

**处理流程**：
```
1. 读取 context/rolling_context.md（如果不存在则创建）
2. 追加新内容，格式化为：
   ## YYYY-MM-DD
   - 学习 MCP 协议
   - 搭建 Hugo 博客
   - 运动30分钟
3. 保存回文件
4. 提示用户："已追加到 rolling context"
```

**优势**：
- ✅ 低门槛（随时记录）
- ✅ 累积效应（滚动上下文）
- ✅ 无需结构化（AI 后期处理）

---

### 2. `/cos review` - 深度对话

**作用**：用三大框架分析当前状态，引导深度思考

**触发**：用户主动调用，或周末自动提示

**三大框架**：

#### **框架 1：SWOT 分析**
- **Strengths**（优势）：本周做得好的地方
- **Weaknesses**（劣势）：需要改进的地方
- **Opportunities**（机会）：可以尝试的新方向
- **Threats**（威胁）：面临的挑战和风险

#### **框架 2：OKR 复盘**
- **Objective**（目标）：本周的核心目标是什么？
- **Key Results**（关键结果）：完成了哪些可量化的成果？
- **Reflections**（反思）：目标设定是否合理？执行中的问题？

#### **框架 3：5 Whys（五问法）**
对本周的关键挑战深入提问：
- 为什么会遇到这个问题？
- 为什么会这样？
- ...（递归 5 次）

**处理流程**：
```
1. 读取 context/rolling_context.md
2. AI 分析内容，识别关键点
3. 依次应用三大框架，生成提问
4. 与用户对话，记录回答
5. 将对话内容追加到 rolling_context.md
6. 提示："复盘完成，可以执行 /cos weekly 生成周报"
```

**示例对话**：
```
AI: 根据本周记录，我注意到你学习了 MCP 协议并搭建了博客。
    让我们用 SWOT 框架深入分析：
    
    【Strengths - 优势】
    1. 你在短时间内完成了技术调研和实践，执行力很强
    2. 选择了 Hugo + MCP 的技术栈，技术前瞻性好
    
    问题：你觉得本周最大的优势是什么？

用户: 我觉得是快速学习能力，能够在1-2天内掌握新概念。

AI: 很好。那么 【Weaknesses - 劣势】呢？
    我注意到运动目标是3次，但本周只完成了2次...
```

---

### 3. `/cos status` - 查看本周状态概览

**作用**：快速查看当前累积的内容和统计

**输出格式**：
```
📊 本周状态概览 (2026-W04)

📝 记录天数：5/7 天
📦 累积条目：23 条
📈 主要类型：
   - 学习：8 条
   - 工作：7 条
   - 生活：5 条
   - 思考：3 条

🔥 高频关键词：
   MCP (5次), Hugo (4次), Skill (3次)

⚠️ 待补充：
   - 周二和周四未记录
   - 运动数据缺失

💡 建议：
   建议在周末前补充缺失的天数，以便生成完整周报。
```

**处理流程**：
```
1. 读取 context/rolling_context.md
2. 统计分析：
   - 记录天数
   - 条目数量
   - 分类统计（学习/工作/生活）
   - 关键词提取
3. 识别缺失：
   - 哪些天没有记录
   - 哪些类型数据不足
4. 生成建议
```

---

### 4. `/cos weekly` - 生成周报

**作用**：基于 rolling_context 生成结构化周报

**前置条件检查**：
```
1. 检查 rolling_context.md 是否有足够数据（至少 3 天记录）
2. 建议用户先执行 /cos review（如果未执行）
3. 确认版本号（v26.4.1）
```

**处理流程**：
```
1. 读取 context/rolling_context.md
2. 读取 references/changelog_template.md（模板）
3. AI 分析与分类：
   - 📦 本周新增：新学习的技能、知识
   - 🔧 优化改进：改进的习惯、方法
   - 🐛 问题修复：解决的问题、克服的挑战
   - ⚠️ 已知问题：未完成的任务、待改进
   - 📊 本周数据：统计数据
4. 生成 Release Notes
5. 保存到 context/changelog.md
6. 清空 rolling_context.md（归档到 archive/）
7. 输出周报给用户审阅
```

**输出示例**：
```markdown
---
title: "人生 v26.4.1 - 2026年第4周发布"
date: 2026-01-27
version: "v26.4.1"
...
---

## 📦 本周新增 (New Features)

- ✅ 学习了 MCP 协议的核心概念
  - 理解了 Client-Server 架构
  - 掌握了 Skill 开发流程
- ✅ 搭建了 Hugo 博客的 life 版块
  - 实现了版本管理系统
  - 配置了 Congo 主题

## 🔧 优化改进 (Improvements)

- 💪 建立了系统化的复盘流程
  - 使用 CoS 架构管理生命周期
  - 采用 Progressive Disclosure 优化上下文

## 🐛 问题修复 (Bug Fixes)

- 🔨 解决了 Hugo 主题的图标显示问题
- 🔨 修复了博客菜单配置错误

## ⚠️ 已知问题 (Known Issues)

- 运动频率不足（2/3次）
- CoS Skill 开发尚未完成

## 📊 本周数据

- 记录天数：5/7
- 学习时长：约 12 小时
- 输出文章：1 篇
- 代码提交：8 commits

## 💭 复盘与思考

### 本周亮点
建立了"人生即软件项目"的管理理念，并通过 CoS 架构实现了系统化记录。

### 遇到的挑战
如何在保持灵活性的同时建立可持续的习惯。

### 下周改进方向
1. 完成 CoS Skill 开发
2. 增加运动频率到3次/周
3. 补充每日记录的完整性

## 🎯 下个版本计划 (v26.5.1)

- [ ] 完成 CoS Skill 开发并测试
- [ ] 优化 rolling context 管理
- [ ] 增加数据可视化
```

---

## 🔬 Rolling Context 原理

### 什么是 Rolling Context？

**定义**：一个动态更新的累积文档，记录当前周期（周）的所有活动。

**类比**：
- **Git Staging Area**：修改还未 commit
- **草稿箱**：内容还在整理中
- **工作内存**：短期记忆，等待转为长期记忆

### 为什么需要 Rolling Context？

#### **问题 1：一次性加载所有历史 → Token 爆炸**
```
传统方式：
用户：生成周报
AI：好的，让我读取你的所有历史记录...
    [加载 100KB 历史数据]
    [Token 用尽，无法生成]
```

#### **解决方案：只累积本周数据**
```
CoS 方式：
用户：/cos log [今天的记录]
AI：追加到 rolling_context.md
    [Token 消耗：~500]

...（重复 7 天）

用户：/cos weekly
AI：读取 rolling_context.md（仅本周数据）
    [Token 消耗：~5000]
    生成周报 ✅
```

### Rolling Context 的生命周期

```
周一 ─────────────→ 周日 ─────────────→ 周一
  │                    │                    │
  ↓                    ↓                    ↓
创建                累积                 清空
rolling_context.md   (每天追加)          (归档到 archive/)
  │                    │                    │
  └──→ 增长 ──→ 达到阈值 ──→ 生成周报 ──→ 重置
       (0→5KB)                          (0)
```

### 实现细节

**文件内容结构**：
```markdown
# Rolling Context - Week 4, 2026

## 2026-01-21 (周一)
- 学习了 MCP 协议基础
- 阅读《Deep Work》第1章
- 运动：跑步 5km

## 2026-01-22 (周二)
- 搭建 Hugo 博客
- 研究 Skill 开发流程

## 2026-01-23 (周三)
...

---
## 元数据
- 记录天数：5/7
- 累积条目：23
- 文件大小：~4.5KB
```

**自动归档机制**：
```python
def weekly_generate():
    # 1. 读取 rolling_context.md
    content = read_file("context/rolling_context.md")
    
    # 2. 生成周报
    release_notes = generate_release(content)
    
    # 3. 保存周报
    save_file("context/changelog.md", release_notes)
    
    # 4. 归档 rolling context
    archive_file("context/rolling_context.md", 
                 f"archive/{year}-W{week}.md")
    
    # 5. 清空 rolling context
    save_file("context/rolling_context.md", "# Rolling Context - Week 5, 2026\n")
```

---

## 📚 学术支撑（是的，有论文！）

### 1. **SimpleMem: Efficient Lifelong Memory for LLM Agents** (2025)

**核心思想**：
- **问题**：LLM 的固定上下文窗口限制了长期记忆
- **方案**：三阶段管道
  1. **Semantic Structured Compression**（语义结构化压缩）
  2. **Recursive Memory Consolidation**（递归记忆整合）
  3. **Adaptive Query-Aware Retrieval**（自适应检索）

**与 CoS 的联系**：
- Rolling Context = 短期记忆（本周）
- Changelog = 长期记忆（历史周报）
- /cos weekly = Memory Consolidation（整合）

### 2. **Claude-Mem: Progressive Disclosure** (2025)

**核心思想**：
- **问题**：传统 RAG 一次性加载所有上下文 → 污染注意力
- **方案**：渐进式披露
  - Layer 1: Index (~1K tokens)
  - Layer 2: Details (按需)
  - Layer 3: Deep Dive (必要时)

**与 CoS 的联系**：
- Level 1: Skill Metadata
- Level 2: Skill Body
- Level 3: Bundled Resources (references/, context/)

### 3. **Context Window Management Strategies** (2025)

**常见策略**：
- Sliding Window（滑动窗口）
- Hierarchical Memory（分层记忆）
- Selective Injection（选择性注入）

**CoS 采用**：
- ✅ Hierarchical Memory（rolling → changelog → archive）
- ✅ Selective Injection（/cos status 只显示概览）

---

## 💡 关键改进点

### 原方案 → CoS 架构

| 改进项 | 原方案 | CoS 架构 | 优势 |
|--------|--------|----------|------|
| **Token 效率** | 一次性加载所有数据 | Progressive Disclosure | ✅ 节省 90% token |
| **灵活性** | 单一命令 `/life-release` | 4 个子命令组合 | ✅ 可按需使用 |
| **可维护性** | 单一 SKILL.md | 模块化文件结构 | ✅ 易于扩展 |
| **用户体验** | 周末一次性输入 | 每天轻量记录 | ✅ 降低门槛 |
| **数据持久化** | 无状态 | Rolling Context | ✅ 累积效应 |

---

## 🚀 实施计划

### Phase 1: 核心 Skill 开发（1小时）

1. **创建目录结构**
   ```bash
   mkdir -p ~/.claude/skills/cos/{references,context/daily,rss-digest}
   ```

2. **编写 SKILL.md**
   - 定义 4 个子命令
   - 配置 Progressive Disclosure

3. **创建参考文件**
   - `references/cos_instruction.md`：完整角色定义
   - `references/changelog_template.md`：周报模板

4. **初始化 rolling_context.md**

### Phase 2: 测试与迭代（0.5小时）

1. 测试 `/cos log`：追加内容
2. 测试 `/cos status`：查看概览
3. 测试 `/cos review`：深度对话
4. 测试 `/cos weekly`：生成周报

### Phase 3: 优化与扩展（未来）

1. **数据可视化**
   - 生成周报时自动生成图表
   - 展示历史趋势

2. **智能提醒**
   - 周三提示："本周已过半，记得记录"
   - 周六提示："建议执行 /cos review"

3. **多模态输入**
   - 支持语音转文字
   - 支持图片 OCR

---

## 📊 成本效益重新评估

### Token 消耗对比

| 操作 | 原方案 | CoS 架构 | 节省 |
|------|--------|----------|------|
| 每日记录 | N/A | ~500 tokens | N/A |
| 查看状态 | N/A | ~1000 tokens | N/A |
| 深度复盘 | 一次性 10K | 增量 2-3K | 70% ↓ |
| 生成周报 | 15-20K | 5-8K | 60% ↓ |

### 时间成本

| 阶段 | 时间投入 |
|------|---------|
| 调研 v1 | 2小时 |
| 调研 v2（CoS） | 1小时 |
| 开发 | 1小时 |
| 测试 | 0.5小时 |
| **总计** | **4.5小时** |

---

## 🎯 总结

### CoS 架构的核心优势

1. **渐进式披露**：按需加载，节省 token
2. **模块化设计**：职责清晰，易于维护
3. **累积效应**：Rolling Context 降低记录门槛
4. **理论支撑**：基于学术研究（SimpleMem、Claude-Mem）

### 与原方案的互补

- 原方案：适合"一次性生成"场景
- CoS 架构：适合"日常管理"场景

**最佳实践**：两者结合
- 日常使用 CoS 架构（/cos log 记录）
- 特殊情况用原方案（临时生成周报）

---

## 下一步

**你现在可以选择**：

1. **立即开发 CoS Skill**（基于新架构）
2. **先开发简化版**（原方案，快速验证）
3. **讨论更多细节**（比如 /cos review 的三大框架是否合理）

**我的建议**：先开发 CoS 架构，因为：
- ✅ 更符合长期使用习惯
- ✅ 有学术理论支撑
- ✅ 开发成本差不多（1小时）

你觉得呢？
