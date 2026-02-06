# `/life-release` Skill 开发方案

**目标**：开发一个可复用的 Claude Skill，自动化生成人生版本管理系统的周报。

---

## 📋 调研总结

### 1. Claude Skill 基础架构

**Skill 文件结构**：
```
.claude/
└── skills/
    └── life-release/
        ├── .claude-skill.yaml    # 核心配置文件
        ├── reference/            # 参考资料（可选）
        │   └── template.md      # 周报模板
        └── examples/            # 示例（可选）
            └── example-input.md
```

**核心文件格式**（`.claude-skill.yaml`）：
```yaml
---
name: life-release
description: 生成人生版本管理系统的周报。分析用户提供的每日记录，识别成长点、学习成果和关键事件，生成结构化的 Release Notes。适用于周日晚或周一早自动生成周报草稿。
---

# Life Release Notes Generator

## 作用
将用户一周的碎片化记录（TODO、笔记、学习日志）整理成结构化的周报（Release Notes），采用软件版本发布的风格。

## 触发时机
- 用户明确说"生成本周周报"或"/life-release"
- 用户提供了一周的数据并询问如何总结

## 输入要求
- 用户提供的本周数据（可以是任意格式的文本）
- 可选：周数和年份（默认使用当前周）

## 执行步骤

### 1. 数据收集
- 读取用户提供的文本内容
- 如果用户指定了 `data/daily/` 目录，使用 Filesystem MCP 读取本周的 Markdown 文件

### 2. 数据分析
对输入内容进行以下分析：

#### 2.1 学习成果识别
- 提取新学到的技能、知识点、概念
- 识别完成的学习任务（阅读、课程、实践）
- **输出格式**：📦 本周新增 (New Features)

#### 2.2 改进点识别
- 识别优化的习惯、方法、工作流程
- 发现效率提升、工具改进
- **输出格式**：🔧 优化改进 (Improvements)

#### 2.3 问题解决识别
- 识别解决的技术问题、bug
- 克服的个人挑战（拖延、焦虑等）
- **输出格式**：🐛 问题修复 (Bug Fixes)

#### 2.4 待改进项识别
- 识别未完成的任务
- 发现的新问题或挑战
- **输出格式**：⚠️ 已知问题 (Known Issues)

#### 2.5 数据统计
- 统计任务完成率、学习时长、输出内容等
- **输出格式**：📊 本周数据

### 3. 版本号生成
- 计算当前周数（例如 2026年第4周）
- 生成版本号：`v{year后两位}.{周数}.1`
- 示例：`v26.4.1`

### 4. 复盘提问（可选）
生成 3-5 个深度复盘问题，引导用户思考：
- "本周最大的成就是什么？为什么？"
- "遇到的最大挑战是什么？如何应对的？"
- "有什么意外的收获或教训？"
- "下周最重要的目标是什么？"
- "如果重来一次，会做什么调整？"

### 5. 生成 Release Notes
按以下模板生成 Markdown 文件：

```markdown
---
title: "人生 v{version} - {year}年第{week}周发布"
date: {current_date}
version: "v{version}"
week: {week}
year: {year}
draft: false
categories: ["人生版本"]
tags: ["周报", "复盘", "成长"]
series: ["Life Changelog"]
---

## 📦 本周新增 (New Features)

- ✅ [从数据中提取的学习成果]
- ✅ [新掌握的技能]

## 🔧 优化改进 (Improvements)

- 💪 [改进的习惯或方法]
- 📝 [效率提升的措施]

## 🐛 问题修复 (Bug Fixes)

- 🔨 [解决的问题]
- 🔨 [克服的挑战]

## ⚠️ 已知问题 (Known Issues)

- [待改进的地方]
- [未完成的任务]

## 📊 本周数据

- 完成任务：X/Y (Z%)
- 学习时长：X 小时
- 输出文章：X 篇
- 代码提交：X commits

## 💭 复盘与思考

### 本周亮点
[AI 生成的总结或等待用户填写]

### 遇到的挑战
[AI 生成的总结或等待用户填写]

### 下周改进方向
[AI 生成的建议]

## 🎯 下个版本计划 (v{next_version})

- [ ] [下周目标1]
- [ ] [下周目标2]
- [ ] [下周目标3]
```

### 6. 输出与保存
- 将生成的内容输出给用户
- 建议用户保存到 `data/weekly/v{version}.md`
- 提示用户审阅后可复制到 `content/life/releases/v{version}.md`

## 约束与边界

### 必须做的
- 保持客观，基于用户提供的数据
- 使用软件 Release Notes 的风格和语言
- 所有分类必须有实际内容支撑，不能凭空捏造
- 版本号必须遵循 `vYY.WW.P` 格式

### 不要做的
- 不要过度解读或臆测用户的想法
- 不要添加用户数据中没有的内容
- 不要使用过于夸张的语言
- 如果某个分类没有内容，直接标注"本周无相关内容"

### 特殊处理
- 如果用户数据很少（少于50字），提示"数据不足，建议补充更多内容"
- 如果用户未指定周数，自动使用当前周
- 如果用户要求修改，仅修改指定部分

## 依赖工具

### 必需
- **Filesystem MCP**（如果读取本地文件）
  ```json
  "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/narwal/knowledge-base/data"]
  }
  ```

### 可选
- **Git MCP**（如果需要统计代码提交）
- **Context7 MCP**（如果需要搜索最佳实践）

## 测试用例

### 测试用例 1：标准流程
**输入**：
```
本周完成了以下内容：
1. 学习了 MCP 协议
2. 搭建了 Hugo 博客的 life 版块
3. 修复了图标显示问题
4. 阅读了《Deep Work》第3章
5. 运动2次（目标3次）
```

**预期输出**：包含所有分类的完整周报

### 测试用例 2：数据不足
**输入**：
```
本周学习了 Python。
```

**预期输出**：提示"数据不足，建议补充更多内容"

### 测试用例 3：修改请求
**输入**：
```
之前生成的周报中，"本周新增"部分加上"完成了技术路线图规划"
```

**预期输出**：仅修改指定部分

## 版本迭代计划

### v1.0（MVP）
- ✅ 基础的数据分析和周报生成
- ✅ 标准模板输出
- ✅ 版本号自动生成

### v1.1（计划中）
- [ ] 自动读取 `data/daily/` 目录
- [ ] 支持多种输入格式（Notion导出、钉钉文档等）
- [ ] 生成数据可视化（图表）

### v2.0（未来）
- [ ] 智能识别情绪和状态
- [ ] 自动生成改进建议
- [ ] 历史对比分析

---

## 实施步骤

### 第 1 步：创建 Skill 目录结构
```bash
mkdir -p .claude/skills/life-release/reference
mkdir -p .claude/skills/life-release/examples
```

### 第 2 步：编写 `.claude-skill.yaml`
复制上述完整的 YAML + Markdown 内容到文件。

### 第 3 步：创建参考模板
将周报模板保存到 `reference/template.md`。

### 第 4 步：创建示例输入
创建 `examples/example-input.md` 包含测试数据。

### 第 5 步：打包 Skill
```bash
cd .claude/skills
zip -r life-release.zip life-release/
```

### 第 6 步：上传到 Claude
- Claude Desktop: Settings > Skills > Upload
- Claude Code: 直接放在 `.claude/skills/` 目录

### 第 7 步：测试
```
用户: 生成本周周报
[粘贴测试数据]
```

---

## 预估工作量

### 调研（已完成）：2小时
- ✅ 研究 Claude Skill 官方文档
- ✅ 分析现有 Release Notes Generator Skill
- ✅ 确定 MCP 工具链

### 开发：1小时
- 编写 `.claude-skill.yaml`（30分钟）
- 创建参考资料和示例（20分钟）
- 打包和上传（10分钟）

### 测试与迭代：0.5-1小时
- 使用真实数据测试（30分钟）
- 优化 Prompt 和模板（30分钟）

**总计**：约 1-2 小时开发 + 测试

---

## 下一步行动

1. **立即开始**：创建 `.claude/skills/life-release/` 目录结构
2. **编写核心文件**：`.claude-skill.yaml`
3. **准备测试数据**：你本周的真实记录
4. **上传并测试**：验证效果
5. **迭代优化**：根据实际使用调整 Prompt

---

**准备好开始开发了吗？** 我可以立即为你生成完整的 Skill 文件！
