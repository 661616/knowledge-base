# 人生版本管理系统 - 数据目录

## 目录结构

```
data/
├── daily/          # 每日记录
│   ├── template.md # 模板
│   └── YYYY-MM-DD.md
└── weekly/         # 周报草稿（AI 生成）
    └── vYY.WW.P.md
```

## 使用流程

### 1. 每日记录（灵活）

你可以用任何方式记录每日内容：
- 钉钉文档
- Notion
- Obsidian
- 纸质笔记
- 语音备忘录

**只需要在周末把这些内容整理后提供给 AI Skill 即可。**

### 2. 生成周报（每周日）

使用 `/life-release` Skill：
```
/life-release --week=4 --year=2026
```

Skill 会：
1. 读取你提供的本周数据
2. AI 分析总结
3. 生成 `data/weekly/v26.4.1.md` 草稿
4. 提出复盘问题

### 3. 审阅与发布

1. 查看并修改 `data/weekly/v26.4.1.md`
2. 复制内容到 `content/life/releases/v26.4.1.md`
3. 或使用 `hugo new life/releases/v26.4.1.md` 并粘贴内容
4. 提交到 Git 并部署

## 版本号规范

`vMAJOR.MINOR.PATCH`

- **MAJOR**: 年份后两位（26 = 2026年）
- **MINOR**: 当年第几周（1-52）
- **PATCH**: 该周的迭代次数（通常为1）

示例：
- `v26.4.1` = 2026年第4周第1次发布
- `v26.52.2` = 2026年第52周第2次发布（特殊情况）

## 周报模板结构

- 📦 本周新增 (New Features)
- 🔧 优化改进 (Improvements)
- 🐛 问题修复 (Bug Fixes)
- ⚠️ 已知问题 (Known Issues)
- 📊 本周数据
- 💭 复盘与思考
- 🎯 下个版本计划

## Tips

- 每日记录不需要完美，碎片化也可以
- 周报生成后可以大幅修改，AI 只是辅助
- 建议先在本地测试几周再公开发布
- 可以保留私密版本（不发布到博客）
