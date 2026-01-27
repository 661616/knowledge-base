+++
date = '2026-01-27T17:07:05+08:00'
draft = false
title = 'Vibe Coding 实践：殊途同归的 Skill、MCP、Prompt、Rules'
categories = ['科技']
tags = ['vibe-coding',"python","AI","Agent","Prompt Engineer"]
+++
# Vibe Coding 实践：殊途同归的 Skill、MCP、Prompt、Rules

## 1. Vibe Coding - 我遇见了难题

### 背景：AI 诊断系统的幻觉问题

我们开发了一个机器人自动化测试的 AI 诊断系统，使用 DeepSeek-V3 分析测试失败原因。但在多次实验后，反复出现**幻觉问题**：

**问题表现**：
- ❌ 编造版本号、套件名、设备 ID
- ❌ 混淆不同测试案例的数据
- ❌ 过度推测底层技术细节（"建图模块未启动"、"传感器数据异常"）
- ❌ 给出不懂的固件级建议

**典型案例**：
```markdown
# AI 输出（错误）
固件未生成建图报告，可能是：
- 建图模块未启动
- 传感器数据异常导致建图中断
- 状态机未正确切换任务模式

问题：我们只是测试团队，看不到传感器数据，不懂固件状态机！
```

**业务澄清**：
```
建图报告 = 从 APM 平台获取的报告数据（应用层）
建图任务 = 机器人执行的实际动作（固件层）

报告为空 → 任务可能未正常执行 → 应该建议"查看 APM 日志"
而不是推测固件内部的模块、传感器、状态机问题
```

### 根本原因：Prompt 被污染

**V1 架构（失败）**：
```python
system_prompt = """
你是资深机器人测试诊断专家。

禁止编造数据...

常见错误知识库：
1. 网络问题：SSH 连接失败...示例：192.168.1.100
2. 框架问题：框架未安装...示例：v1.0.0
3. 固件问题：建图模块未启动...

示例1：套件 A，设备 001，版本 1.0.0...
示例2：套件 B，设备 002，版本 1.1.0...

输出格式：...
"""

user_prompt = f"""
请诊断：套件 C，设备 003，版本 1.2.0...
"""
```

**问题**：
1. **上下文污染**：AI 分不清示例和实际数据，容易"复制"示例中的 192.168.1.100、v1.0.0
2. **规则淹没**：禁令规则埋在长文本中，容易被忽略
3. **记忆混淆**：多次对话后，历史案例干扰当前诊断
4. **知识泛化**：AI 看到"建图模块"的示例，就会过度推测

## 2. 从长 System Prompt 到分层架构

### 核心改进思路

**问题本质**：信息边界不清晰

**解决方案**：分层架构 + 明确标注

### V2 架构（成功）

```python
# 第一层：System Message - 核心规则
system_prompt = """
你是机器人测试诊断系统。

## 🚨 最高优先级规则

### 禁止编造数据
- 严禁编造版本号、套件名、设备 ID
- 字段为空就说"无此数据"

### 禁止过度推测（最重要）
- 只描述看到的现象
- 不要推测底层原因（固件、传感器、状态机）
- 不要给出不确定的技术建议

✅ 正确：建议查看 APM 日志
❌ 错误：建图模块未启动

## 业务概念区分
建图报告 = 从 APM 获取的数据
建图任务 = 机器人执行的动作
报告为空 → 查看 APM 日志（不要推测任务细节）
"""

# 第二层：User Message 1 - 知识库（明确标注）
knowledge_base = """
# 参考知识库

以下是常见失败模式，供你参考：

⚠️ 重要：这些是参考模式，不是当前案例的数据
⚠️ 使用这些模式来识别问题类型，但必须引用实际日志作为证据
⚠️ 不要直接复制示例中的数据

1. 网络问题示例：
   症状：[ERROR] SSH Connection refused
   判断：网络连通性问题
   建议：检查网络配置

2. 建图报告为空示例：
   症状：[EXCEPTION] 建图报告为空
   判断：任务可能未正常执行
   建议：查看 APM 日志
"""

# 第三层：Assistant - 确认隔离
assistant_confirm = "知识库已加载，我将基于这些模式进行诊断。"

# 第四层：User Message 2 - 实际数据（清晰分隔）
actual_data = """
# 诊断任务

这是一个全新的、独立的诊断任务，与之前的案例无关。

## 基本信息
- 测试套件：【冒烟测试】J64/J65-MR536
- 设备 ID：1b56d4e293e74bf2b188afbf330580a5
- 固件版本：v00.04.10.224

## 失败日志
[EXCEPTION] 建图报告为空
[EXCEPTION] 【校验点】等待当前任务结束: 当前:【空闲】IN_STATION
"""

# 组装
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": knowledge_base},
    {"role": "assistant", "content": assistant_confirm},
    {"role": "user", "content": actual_data}
]
```

### 关键改进点

| 维度 | V1 | V2 |
|------|----|----|
| **结构** | 单一长 Prompt | 分层消息 |
| **知识库标注** | 无标注 | "这是参考，不是实际数据" |
| **会话隔离** | 无 | "全新独立任务" |
| **禁令强度** | 一句话 | 最高优先级 + 示例对比 |
| **业务知识** | 隐含 | 明确区分概念 |

### 效果

```markdown
# V1 输出（错误）
固件未生成建图报告，可能是：
- 建图模块未启动
- 传感器数据异常

# V2 输出（正确）
现象：建图报告为空，机器人状态为空闲
排除：网络正常、框架正常
下一步：查看 APM 应用日志，确认建图任务执行情况
```

## 3. 分层架构 == Skill？什么是 Skill

### 发现：这就是 Claude Skill 的思想！

在解决问题后，我发现这个分层架构和 **Claude Skill** 的设计理念高度一致。

### 什么是 Claude Skill？

**定义**：Skill 是 Claude 的一种能力扩展机制，通过文件夹组织指令、脚本和资源，让 AI 能够动态加载专门的能力。

**文件结构**：
```
skill-name/
├── SKILL.md (必需)
│   ├── YAML frontmatter（元数据）
│   │   ├── name: (必需)
│   │   └── description: (必需)
│   └── Markdown 指令（核心内容）
└── 资源（可选）
    ├── scripts/          - 可执行脚本
    ├── references/       - 参考文档
    └── assets/           - 资源文件
```

**SKILL.md 示例**：
```markdown
---
name: robot-test-diagnosis
description: 诊断机器人自动化测试失败的根本原因。当用户提供测试报告、执行日志、失败用例信息时使用。
---

# 机器人自动化测试诊断专家

## 🚨 最高优先级规则

### 禁止编造数据
### 禁止过度推测

## 诊断流程
1. 理解执行链路
2. 定位失败点
3. 分析日志
...

## 输出格式
```

### Skill 的工作原理

**渐进式披露（Progressive Disclosure）**：
```
1. 扫描阶段：Claude 读取所有 Skill 的 YAML frontmatter
2. 匹配阶段：根据 description 判断是否需要该 Skill
3. 加载阶段：如果匹配，加载完整的 Markdown 内容
4. 执行阶段：按照 Skill 的指令执行任务
```

**优势**：
- ✅ 节省 Token（只在需要时加载）
- ✅ 清晰的边界（每个 Skill 独立）
- ✅ 易于维护（修改 MD 文件即可）
- ✅ 可复用（打包成 .skill 文件分发）

### 我的实现：手动 Skill 加载

虽然我用的是 DeepSeek（不支持 Claude Skill 系统），但可以手动实现：

```python
class AIClientV2:
    def __init__(self):
        # 手动加载 Skill 文件
        skill_path = "skills/robot-test-diagnosis"
        self.skill_content = self._load_skill(skill_path)
        self.failure_patterns = self._load_failure_patterns(skill_path)
    
    def _load_skill(self, path):
        skill_file = Path(path) / "SKILL.md"
        content = skill_file.read_text()
        # 提取 Markdown body（跳过 YAML frontmatter）
        return extract_markdown_body(content)
    
    def diagnose(self, context):
        # 手动构建分层消息（模拟 Skill 的加载机制）
        messages = [
            {"role": "system", "content": self.skill_content},
            {"role": "user", "content": f"# 参考知识库\n{self.failure_patterns}"},
            {"role": "assistant", "content": "知识库已加载"},
            {"role": "user", "content": format_diagnostic_data(context)}
        ]
        
        return self.client.chat.completions.create(
            model="deepseek-v3",
            messages=messages,
            temperature=0.3  # 降低随机性
        )
```

**创建的文件结构**：
```
skills/robot-test-diagnosis/
├── SKILL.md                    # 核心指令
└── references/
    └── failure_patterns.md     # 知识库
```

### 核心启示

**Skill 不是特定平台的功能，而是一种思想**：
1. **结构化组织**：用文件夹管理能力
2. **分层加载**：规则、知识库、数据分离
3. **明确边界**：每个部分的作用清晰
4. **可维护性**：修改文件而非代码

## 4. Skill vs MCP vs Rules vs Prompt

### 本质认知

> **它们都是 Prompt**，只是不同平台对**组织和加载方式**的不同实现。

```
┌─────────────────────────────────────┐
│         Prompt (最底层)              │
│  "你是一个专家，禁止编造数据..."      │
└─────────────────────────────────────┘
              ↑
              │ 不同的组织方式
              │
    ┌─────────┴─────────┬─────────────┐
    │                   │             │
┌───▼────┐      ┌──────▼─────┐  ┌───▼────┐
│ Rules  │      │   Skill    │  │  MCP   │
│ (项目) │      │  (能力)    │  │ (工具) │
└────────┘      └────────────┘  └────────┘
    │                   │             │
    └───────────┬───────┴─────────────┘
                │
                ↓
        最终都是 Prompt 传给 LLM
```

### 详细对比

#### Claude Skill

**定位**：可复用的能力包

**特点**：
- 📁 文件系统组织（SKILL.md + resources）
- 🔍 自动发现（根据 description）
- 📦 渐进加载（节省 token）
- 🎯 单一职责（一个 Skill 一个能力）

**适用场景**：
- 特定领域的专业能力（如代码审查、测试诊断）
- 需要复用的工作流
- 团队协作（可分享 .skill 文件）

**示例**：
```
skills/
├── code-reviewer/          # 代码审查 Skill
├── test-diagnosis/         # 测试诊断 Skill
└── api-designer/           # API 设计 Skill
```

#### MCP (Model Context Protocol)

**定位**：跨平台的工具和资源标准

**特点**：
- 🔌 协议标准（不绑定单一 AI）
- 🛠️ 工具 + 资源（不只是 Prompt）
- 🌐 服务化架构（运行时注册）
- 🔄 动态能力（可扩展）

**适用场景**：
- 需要调用外部工具（数据库、API）
- 需要访问资源（文件、网页）
- 跨平台能力（Claude、Cursor、VSCode 等）

**示例**：
```json
// MCP 服务器配置
{
  "confluence": {
    "command": "mcp-server-confluence",
    "tools": ["get_page", "search_page"],
    "resources": ["confluence://page/123"]
  }
}
```

#### Cursor Rules

**定位**：项目级的 AI 配置

**特点**：
- 📋 项目级生效（自动应用）
- 📄 多文件支持（.cursor/rules/*.mdc）
- 🔁 继承机制（alwaysApply）
- 💻 IDE 集成（无缝体验）

**适用场景**：
- 项目级编码规范
- 团队协作约定
- 技术栈特定规则

**示例**：
```markdown
---
alwaysApply: true
---
本项目使用 poetry 管理依赖
禁止在没有显示说明时创建文档
```

#### Prompt

**定位**：最原始的指令文本

**特点**：
- 📝 纯文本（最灵活）
- 🚀 即时生效（无需配置）
- 🔧 完全控制（自由度最高）
- 📱 通用性强（所有 LLM 都支持）

**适用场景**：
- 一次性任务
- 快速实验
- 不需要复用

**示例**：
```python
prompt = "你是一个 Python 专家，帮我优化这段代码..."
```

### 对比表格

| 维度 | Skill (Claude) | MCP | Rules (Cursor) | Prompt |
|------|----------------|-----|----------------|--------|
| **本质** | 结构化 Prompt | 工具协议 | 项目配置 | 纯文本 |
| **文件格式** | MD + YAML | JSON 配置 | MD/YAML | 任意 |
| **加载方式** | 自动发现 | 服务注册 | 自动应用 | 手动传入 |
| **作用域** | 单个能力 | 工具集 | 整个项目 | 单次对话 |
| **可移植性** | Claude 专属 | 跨平台标准 | Cursor 专属 | 通用 |
| **维护成本** | 低（MD 文件） | 中（需要服务） | 低（配置文件） | 高（代码耦合） |
| **灵活性** | 中 | 高 | 低 | 最高 |
| **复用性** | 高 | 高 | 中 | 低 |

### 选择建议

```
用 Skill 当：
  ✓ 需要可复用的专业能力
  ✓ 使用 Claude 平台
  ✓ 团队协作分享

用 MCP 当：
  ✓ 需要调用外部工具
  ✓ 跨平台使用
  ✓ 动态扩展能力

用 Rules 当：
  ✓ 项目级规范
  ✓ 使用 Cursor IDE
  ✓ 自动生效

用 Prompt 当：
  ✓ 一次性任务
  ✓ 快速实验
  ✓ 完全自定义
```

### 混合使用

实际项目中，可以混合使用：

```python
# 1. Cursor Rules：项目级规范
# .cursor/rules/coding.mdc
---
alwaysApply: true
---
使用 poetry 管理依赖
遵循 PEP 8 规范

# 2. Skill 思想：能力模块化
# skills/test-diagnosis/SKILL.md
核心诊断逻辑...

# 3. MCP：外部工具集成
# 使用 Confluence MCP 获取文档

# 4. Prompt：组装一切
system_prompt = load_skill("test-diagnosis")
tools = load_mcp("confluence")
messages = build_messages(system_prompt, tools, user_input)
```

## 5. 最佳实践

### 防幻觉的核心策略

基于本次实践，总结出以下可复用的方法：

#### 1. 分层架构

**原则**：清晰的信息边界

```python
# ✅ 正确的分层
messages = [
    # 第一层：角色和规则
    {"role": "system", "content": "你是XX专家\n\n## 禁令\n..."},
    
    # 第二层：知识库（明确标注）
    {"role": "user", "content": "# 参考知识库\n⚠️ 这是参考，不是实际数据\n..."},
    
    # 第三层：确认隔离
    {"role": "assistant", "content": "知识库已加载，我将基于实际数据诊断"},
    
    # 第四层：实际数据
    {"role": "user", "content": "# 本次任务\n这是全新的独立任务\n..."}
]

# ❌ 错误的混合
system_prompt = """
你是专家
禁令...
知识库：示例1、示例2...
"""  # 全部混在一起，AI 分不清
```

#### 2. 会话隔离

**原则**：每次都是新开始

```markdown
## 强调独立性

这是一个**全新的、独立的**诊断任务。

**忘记**所有之前的案例、示例和对话历史。

只基于**当前提供的数据**进行分析。
```

**效果**：防止历史对话污染当前诊断

#### 3. 禁令强化

**原则**：用视觉标记 + 对比示例

```markdown
## 🚨 最高优先级规则

### 禁止编造数据
- **严禁**编造任何具体数值
- 字段为空就说"无此数据"

### 禁止过度推测
- 只描述看到的现象
- 不要推测底层原因

✅ 正确示例：
- "建图报告为空，建议查看 APM 日志"

❌ 错误示例：
- "建图模块未启动"（你不知道是哪个模块）
- "传感器数据异常"（你看不到传感器数据）

**如果你违反了这个规则，你的诊断将被视为失败！**
```

**技巧**：
- 用 🚨 emoji 吸引注意
- 用"最高优先级"强调重要性
- 用对比示例让 AI 理解边界
- 用"威胁"加强约束

#### 4. 业务知识明确化

**原则**：不要让 AI 猜测概念

```markdown
## 重要业务概念区分

### 建图相关
- **建图任务**：机器人执行的实际建图动作（由固件控制）
- **建图报告**：建图完成后从 APM 平台获取的报告数据
- **关键理解**：
  - 校验"建图报告"是在验证能否从 APM 获取到报告
  - 如果报告为空，通常是建图任务本身没有正常开始/结束
  - **不要推测**建图任务失败的具体原因（传感器、模块等）
  - **正确做法**：建议"查看 APM 应用日志，确认建图任务执行情况"
```

**效果**：AI 不会自己理解错概念

#### 5. 参数优化

```python
# 降低随机性
completion = client.chat.completions.create(
    model="deepseek-v3",
    messages=messages,
    temperature=0.3,        # 0.5 → 0.3，减少"创造性"
    top_p=0.9,             # 限制词汇选择范围
    frequency_penalty=0.3,  # 减少重复
    presence_penalty=0.1    # 鼓励多样性但不过度
)
```

#### 6. 上下文管理

```python
# 日志截断
key_logs = case.error_logs[:30]  # 只取前 30 条

# 精简格式
success_cases = [
    f"- ✅ {case.name}（耗时 {case.duration:.2f}s）"
    for case in context.all_cases if case.status == "success"
]

# 按需加载知识库
report = client.diagnose(
    context, 
    use_knowledge_base=True if is_complex else False
)
```

### 文件组织最佳实践

#### Skill 风格的文件结构

```
project/
├── skills/                      # 能力模块
│   └── robot-test-diagnosis/
│       ├── SKILL.md            # 核心指令（3-5K 字符）
│       └── references/
│           └── failure_patterns.md  # 知识库（5-10K 字符）
│
├── src/
│   └── services/
│       └── ai_client_v2.py     # 加载和组装逻辑
│
└── tests/
    └── test_skill_system.py    # 测试套件
```

**SKILL.md 模板**：
```markdown
---
name: capability-name
description: 简短描述这个能力的用途和使用场景
---

# 能力名称

## 🚨 最高优先级规则
[禁令规则，用 emoji 和对比示例强化]

## 核心职责
[明确的能力范围]

## 业务知识
[概念区分，避免 AI 误解]

## 工作流程
[步骤化的处理逻辑]

## 输出格式
[严格的模板]

## 示例模式
[好的示例 vs 坏的示例]
```

#### 加载逻辑

```python
class SkillBasedClient:
    def __init__(self, skill_path: str):
        # 1. 加载 Skill
        self.skill_content = self._load_skill(skill_path)
        self.references = self._load_references(skill_path)
    
    def _load_skill(self, path: str) -> str:
        """加载 SKILL.md，提取 Markdown body"""
        skill_file = Path(path) / "SKILL.md"
        content = skill_file.read_text(encoding="utf-8")
        
        # 跳过 YAML frontmatter
        lines = content.split("\n")
        body_lines = []
        frontmatter_count = 0
        
        for line in lines:
            if line.strip() == "---":
                frontmatter_count += 1
                continue
            if frontmatter_count >= 2:
                body_lines.append(line)
        
        return "\n".join(body_lines)
    
    def execute(self, task_data: dict) -> str:
        """执行任务"""
        messages = self._build_messages(task_data)
        
        return self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=0.3,
            top_p=0.9,
            frequency_penalty=0.3
        )
    
    def _build_messages(self, task_data: dict) -> list:
        """构建分层消息"""
        return [
            {"role": "system", "content": self.skill_content},
            {"role": "user", "content": f"# 参考知识库\n{self.references}"},
            {"role": "assistant", "content": "知识库已加载"},
            {"role": "user", "content": self._format_task(task_data)}
        ]
```

### 测试和验证

#### 幻觉检测测试

```python
def test_no_hallucination():
    """测试：不编造数据"""
    # 故意提供空数据
    context = DiagnosticContext(
        suite_name="Unknown",
        device_info={},
        ota_info=None,
        failed_cases=[]
    )
    
    report = client.diagnose(context)
    
    # 验证：不应该出现编造的数据
    assert "1.0.0" not in report  # 不应该有编造的版本号
    assert "192.168" not in report  # 不应该有编造的 IP
    assert "无此数据" in report or "数据不足" in report

def test_no_over_speculation():
    """测试：不过度推测"""
    context = build_building_map_failure_context()
    report = client.diagnose(context)
    
    # 验证：不应该推测固件细节
    forbidden_terms = [
        "建图模块未启动",
        "传感器数据异常",
        "状态机未切换"
    ]
    for term in forbidden_terms:
        assert term not in report, f"不应该出现过度推测：{term}"
    
    # 验证：应该给出合理建议
    assert "APM 日志" in report or "嵌入式团队" in report
```

#### 数据准确性测试

```python
def test_data_accuracy():
    """测试：准确引用实际数据"""
    context = build_test_context(
        suite_name="冒烟测试套件",
        device_id="test_device_123",
        version="v1.2.3"
    )
    
    report = client.diagnose(context)
    
    # 验证：必须引用实际数据
    assert "冒烟测试套件" in report
    assert "test_device_123" in report
    assert "v1.2.3" in report
```

### 迭代优化流程

```
1. 收集问题案例
   ↓
2. 分析幻觉类型（编造数据？过度推测？）
   ↓
3. 更新 SKILL.md
   - 强化禁令规则
   - 添加对比示例
   - 明确业务概念
   ↓
4. 运行测试验证
   ↓
5. A/B 测试对比
   ↓
6. 全量上线
```

**关键点**：
- Skill 文件独立 → 修改不需要改代码
- 测试驱动 → 每次优化都有验证
- 渐进式 → 先小范围测试，再全量

### 跨平台适配

即使不使用 Claude，也可以借鉴 Skill 思想：

```python
# 适配 OpenAI
class OpenAISkillClient:
    def __init__(self, skill_path):
        self.skill = load_skill(skill_path)
    
    def chat(self, messages):
        return openai.ChatCompletion.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": self.skill},
                *messages
            ]
        )

# 适配 Anthropic Claude
class ClaudeSkillClient:
    def __init__(self, skill_path):
        self.skill = load_skill(skill_path)
    
    def chat(self, messages):
        return anthropic.Anthropic().messages.create(
            model="claude-3-opus",
            system=self.skill,
            messages=messages
        )

# 适配 DeepSeek
class DeepSeekSkillClient:
    def __init__(self, skill_path):
        self.skill = load_skill(skill_path)
    
    def chat(self, messages):
        return OpenAI(
            api_key=deepseek_key,
            base_url="https://api.deepseek.com"
        ).chat.completions.create(
            model="deepseek-v3",
            messages=[
                {"role": "system", "content": self.skill},
                *messages
            ]
        )
```

### 总结

**核心原则**：
1. **清晰的边界**：分层架构，明确标注
2. **强化的约束**：禁令前置，示例对比
3. **业务对齐**：概念明确，不让 AI 猜
4. **持续优化**：文件独立，易于迭代

**可复用的成果**：
- ✅ Skill 文件结构和模板
- ✅ 分层消息架构
- ✅ 防幻觉的 5 大策略
- ✅ 测试和验证方法
- ✅ 跨平台适配方案

**最终理解**：
> Skill、MCP、Rules、Prompt 本质都是 Prompt，只是组织方式不同。关键不是用哪个平台，而是如何**清晰地组织信息**、**强化约束规则**、**明确业务边界**。

---

*本文档基于实际项目经验总结，代码和配置均可直接使用。*

