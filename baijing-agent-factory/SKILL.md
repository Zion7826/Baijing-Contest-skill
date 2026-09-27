---
name: baijing-agent-factory
agent_created: true
description: 百景大赛（工业AI应用创新挑战）参赛智能体全流程流水线：上传赛题任务书 docx → 自动解析要求 → 构建智能体 Web 应用 → 实测留证 → 产出操作手册/应用案例/参赛报告书。当用户上传比赛任务书并要求"做智能体/做应用/参赛交付/出手册案例报告"时使用。沉淀了跨系统表格桥接智能体项目的全部实战经验（架构原则、环境坑、测试方法论、文档管线）。
---

# 百景大赛智能体流水线（赛题 → 交付）

输入：赛题任务书 docx。产出：可运行的智能体 Web 应用 + 图文操作手册 + 图文应用案例 + 参赛报告书。
参考实现：`D:/WorkBuddy/希望早睡百景/agent_app/`（跨系统表格桥接智能体，verify 3/3、5案例10断言、10万行基准全过）。

## 总流程（五阶段，每阶段先建 TaskCreate 任务跟踪）

```
① 赛题解析 → ② 智能体构建 → ③ 实测留证 → ④ 文档产出 → ⑤ 交付自检
```

---

## 阶段一：赛题解析

1. **提取文本**：docx 用 python-docx 抽全文存 `docx_text.txt`（后续阶段反复引用，不要每次重抽）：
   ```python
   from docx import Document
   text = "\n".join(p.text for p in Document(path).paragraphs if p.text.strip())
   # 表格内容单独抽：for t in doc.tables: for row in t.rows: ...
   ```
2. **提炼四要素**（写进任务清单描述，防遗漏）：
   - 功能要求（逐条编号，后面逐条对照验收）
   - 验收标准（几条就建几个 verify 项）
   - 目标系统/导入模板格式（若涉及"导入老旧系统"，反向实现其列序/列名/类型）
   - 约束（算力、模型、数据规模、赛道主题）
3. **识别唯一缺口思维**：开发中后期用任务书原文逐条 grep 对照已实现功能，缺什么补什么（本项目靠这招发现"多源数据聚合"缺口）。

---

## 阶段二：智能体构建

### 架构铁律（评委与答辩的生命线）
1. **LLM 只做"理解"，绝不做"计算"**：意图识别、字段映射兜底、自由模式规划交给 LLM；一切数值计算/筛选/聚合走 pandas 确定性规则引擎。没配 LLM key 时纯规则离线可跑（演示不翻车）。
2. **新业务 = 加一个 Skill 子类**，不动主流程（`core/rules.py` 里 `Skill` 基类 + `SKILLS` 注册表）。
3. **意图路由 LLM 优先**：关键词匹配只做回退（"工时"这类词会误伤其他指令）。
4. **异常永远留痕不静默**：脏数据隔离进异常日志（行号+原始值），解析/计算阶段都记。

### 目录结构模板
```
agent_app/
├── core/          # parser(解析+类型推断+异常) fields(字段词典+匹配)
│                  # filters(条件解析+build_mask) mapper rules(技能+引擎)
│                  # llm(OpenAI格式客户端) agent(意图路由+编排) exporter
├── webapp/        # server.py(FastAPI) static/{index.html,style.css,app.js}
├── test_cases/    # run_cases.py bench_large.py enhance_test.py
├── data/demo/     # make_demo.py 生成的演示数据
├── docs/          # 手册/案例 docx + 报告书副本
└── verify.py      # 对照验收标准的自检入口
```

### Web 技术栈
- FastAPI + uvicorn（端口 8620）+ 原生 JS 前端（无框架，部署零依赖）
- 设置页保存 `llm_config.json`（OpenAI 格式：base_url/api_key/model），测试连接按钮
- 界面去 AI 味：企业蓝 `#2456C8`、6px 圆角、**无 emoji**、真实业务文案、图例/标签/表格斑马纹

### LLM 接入（可选但有则加分）
- 优先问用户要 OpenAI 格式 API；没有则部署 `workbuddy2api-hub` 网关反代（端口 8788，`WB_PROXY_DEFAULT_REALM=cn`，用户自行 OAuth 绑定）。
- LLM 调用带 3 次重试 + timeout 90s（30s 会偶发超时）；JSON 输出解析失败要重试，不能静默吞。

### 环境速查（本机实测）
| 项 | 值 |
|---|---|
| Python | `C:/Users/Zion7826/.workbuddy/binaries/python/envs/default/Scripts/python.exe` |
| pip | 必加 `-i https://mirrors.aliyun.com/pypi/simple/`（清华/官方源不通） |
| Node | `C:/Users/Zion7826/.workbuddy/binaries/node/versions/22.22.2-3/node.exe` |
| 端口 | Web 8620 / 网关 8788 / Streamlit演示 8601 |
| **代理坑** | 本机有 `HTTP_PROXY=127.0.0.1:18939`，会劫持发往 127.0.0.1 的请求致 502。**httpx 一律 `trust_env=False`；启动服务时清空代理变量**（`HTTP_PROXY= HTTPS_PROXY= ... uvicorn ...`） |

---

## 阶段三：实测留证（答辩弹药库）

1. **verify.py**：对照验收标准逐条自检，输出 PASS/FAIL。改核心代码后必跑。
2. **多案例实测**（≥5 个，覆盖面即说服力）：标准场景 / 异构中文表头 / 纯英文 CSV / 千行脏数据（注入 `!!!`、`###`、空行）/ 自由语言描述全新领域表。每案例带确定性断言（人工复算期望值），随机数据要固定种子或用确定性公式。
3. **性能基准**：10 万行级生成→解析→执行→抽样复算，记录耗时（目标：解析 <10s、计算 <5s）。
4. **截图留证**：playwright-core 跑 `take_shots.js` 截 10 张图（登录/上传/执行/结果/异常/验收/设置各界面），存 `docs_assets/web/`。
   ```bash
   NODE_PATH=<node workspace>/node_modules <node> take_shots.js
   ```
5. **测试脚本的代理免疫**：所有 httpx 调用 `trust_env=False`，否则本地回环请求走代理全挂。
6. 测试中的 LLM 相关断言（引号变体、空 JSON、措辞漂移）容易偶发失败——修法：解析层加引号剥离/元词黑名单，决策层空结果重试，**不要把断言改松来糊弄**。

---

## 阶段四：文档产出

### 4a. 图文操作手册 + 应用案例（HTML → docx 管线）
1. HTML 排版（企业风模板，与界面同色系），截图用阶段三产物。
2. 转 docx：**转换依赖必须装进自有 venv**（插件 venv 会被反复重建），用 `PYTHONPATH` 跑 `html_to_docx`：
   ```bash
   PYTHONPATH=<插件路径> <自有venv python> html_to_docx.py in.html out.docx
   ```
3. 手册结构：快速上手 → 界面导览（每界面一图）→ 核心功能分步 → 常见问题。案例结构：业务背景 → 数据说明 → 自然语言指令 → 过程截图 → 结果复算验证。

### 4b. 参赛报告书
**调 `baijing-report-fill` skill**（配合 `docx-local-edit-gotchas`）。要点回顾：
- 原模板永不动，`cp` 工作副本；edsdk 单次执行铁律（open→编辑→save 一次 Bash 内完成）
- 内容只写实测过的东西（引用阶段三数字），封面身份字段留用户
- 若任务书要求"交互式对话或配置界面"，报告里要写清两条都做了+实测场景

### 4c. 答辩要点预置
- "怎么证明能导入目标系统"→ 模板格式逐列对齐 + 异常日志表（从任务书要求反向实现）
- 性能数字全部有实测记录可现场演示
- 架构一句话："模型负责理解，引擎负责计算，异常全程留痕"

---

## 阶段五：交付自检清单

- [ ] 任务书功能逐条对照：无缺口（grep 任务书关键词 vs 代码/文档）
- [ ] verify.py 全 PASS；5+ 案例断言全过；基准数据已记录
- [ ] 服务在线可演示（告诉用户地址与端口），重启后能自恢复
- [ ] 操作手册/案例 docx 图文齐全；报告书副本已填、原模板未动
- [ ] 封面身份字段已提醒用户补填
- [ ] present_files 交付全部文档 + 应用地址

## 常见中断恢复（电脑重启/服务挂）
1. 网关：`cd workbuddy2api-hub && WB_PROXY_DEFAULT_REALM=cn <venv python> wb_proxy.py`（后台）
2. Web：`cd agent_app/webapp && HTTP_PROXY= HTTPS_PROXY= <venv python> -m uvicorn server:app --host 127.0.0.1 --port 8620`
3. `curl --noproxy "*"` 健康检查两个服务后再继续。
