# Baijing-Contest Skill 百景大赛参赛技能包

面向「工业AI应用创新挑战·百景大赛」的智能体参赛技能集合，配合 WorkBuddy 类 AI 编码助手使用。

## 包含的 Skill

### 1. baijing-agent-factory（总控流水线）
上传赛题任务书 docx → 自动解析要求 → 构建智能体 Web 应用 → 实测留证 → 产出操作手册/应用案例/参赛报告书的五阶段全流程。

覆盖：
- 赛题解析方法论（功能要求/验收标准/模板格式/约束四要素提炼）
- 智能体架构铁律（LLM 只做理解、确定性引擎做计算、Skill 子类扩展）
- 实测留证方法论（verify 自检、多案例断言、10 万行级基准、截图）
- 环境速查（Python venv / pip 镜像 / 代理坑 / 端口约定）

### 2. baijing-report-fill（报告书填写）
保持原 docx 模板格式，往提纲模板里填章节正文 + 插界面截图的完整流程与避坑手册（editor_sdk 单次执行铁律等血泪经验）。

## 使用方式

把对应目录放进 AI 助手的 skills 目录（如 `~/.workbuddy/skills/`），助手即可在相关任务中自动加载。

## 配套项目

- 智能体本体：[Cross-System-Spreadsheet-Data-Automation-Bridge-and-Processing-Agent](https://github.com/Zion7826/Cross-System-Spreadsheet-Data-Automation-Bridge-and-Processing-Agent)

## 开源许可

GPL-3.0，Copyright (c) 2026 Zion7826，详见 [LICENSE](LICENSE)。
