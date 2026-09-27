---
name: baijing-report-fill
agent_created: true
description: 百景大赛（东莞工业AI应用创新挑战）参赛报告书填写流程。当用户要求填写百景大赛报告书模板、或任何"保持原 docx 格式往模板里填内容+插截图"的比赛报告任务时使用。核心：tencent-local-office-edit edsdk.py 的「单次执行铁律」（open→编辑→save 必须在 ONE 次 Bash 调用内完成，实例跨调用必死）、从后往前填充、实测数据留证、封面身份字段留给用户。
---

# 百景大赛报告书填写流程

## 何时使用
- 用户给出比赛报告书 docx 模板（提纲式占位），要求"写报告书，不要改变格式"。
- 任何「往既有 docx 模板填章节正文 + 插界面截图」的任务。

## 铁律（先读这里，血泪换来的）

1. **原模板永不动**：先 `cp` 到工作区副本（如 `agent_app/docs/报告书_工作副本.docx`），所有编辑只碰副本。交付时明确告诉用户原文件在哪、副本在哪。
2. **单次执行铁律**：editor_sdk 实例**跨 Bash 调用极易死**（`get_pool_status` 里只剩 `file_path=""` 的僵尸条目，内存编辑全丢）。所以 **open → 轮询就绪 → 全部编辑 → save 必须在 ONE 次 Bash/子进程脚本调用内完成**。绝不要把"打开"和"编辑"拆成两次 Bash。
3. **fresh open 的 file_id 就是文件完整路径字符串**（不是 UUID）。打开后不要依赖旧会话的 file_id/table_id。
4. **open_file 是异步流式加载**：调用后必须轮询 `doc_find` 直到锚点命中（每次 sleep 2s，最多 30 次），插了图片后**再额外 sleep 8s** 等图片加载，否则 structure 取不到节点。
5. **table_id 每次打开都会变**：填封面表前现查 `doc_list_tables`，拿 `(tables[0]).table_id`。
6. **save/open 的响应可能是纯文本不是 JSON**：解析失败 ≠ 保存失败。判断保存成功要用「重新打开后 `doc_resolve_document_structure` 核对内容」来确认，不要只看响应格式。

## 工作流（六步）

### 第 0 步：备料
- `cp` 模板 → 工作副本。
- 读取模板结构：`doc_resolve_document_structure`（mode=full）导出 JSON，人工梳理：封面表位置、目录页、各章标题与占位说明段。
- 收集**实测证据**：界面截图路径、验收结果、性能数字。**内容只写实测过的东西，不编造**——答辩会被追问。

### 第 1 步：定位锚点（doc_find）
```python
def find(text):
    res = call("doc_find", {"file_id": FID, "text": text})
    locs = res.get("locations") or []   # ← 返回键是 locations（含 begin/end）
    return (int(locs[0]["begin"]), int(locs[0]["end"])) if locs else (None, None)
```
- **structure 的 text_preview 会截断到 ~7 字**，节点匹配一律用短前缀（如 `"作品的主要"`），别用全称。
- `doc_find` 匹配文本要按文档实际写法；全角括号「（」可能影响命中，找不到就去掉括号重试。

### 第 2 步：填章节正文（从后往前）
- 每章：`doc_find` 找标题 → 删标题后的占位说明段 → 在标题 end 处 `doc_insert_paragraph_with_text` 逐段插正文。
- **按文档位置从后往前处理**（先第 5 章后第 1 章），前面的锚点坐标不漂移。
- `doc_insert_paragraph_with_text` 返回 `{"range": {"begin":X,"end":Y}, "next_index": N}`，链式插入用 `next_index`。

### 第 3 步：插截图
```python
p = call("doc_insert_paragraph_with_text", {"file_id": FID, "idx": idx, "text": ""})
img_idx = p["range"]["begin"]                      # range 是 {begin,end} 字典
call("doc_insert_image", {"file_id": FID, "idx": img_idx,
                          "image_path": img_path, "w": 440, "h": 248})
cap = call("doc_insert_paragraph_with_text",
           {"file_id": FID, "idx": p["next_index"],
            "text": f"图N {caption}", "paragraph_property": {"jc": "center"}})
```
- **每个图之间会多出一个空段**，插完统一清理（见第 5 步）。
- 图注格式 `图N xxx` 便于后续 doc_find 定位与验收。

### 第 4 步：填封面表
```python
tabs = call("doc_list_tables", {"file_id": FID})
tid = (tabs.get("tables") or [{}])[0].get("table_id")   # 现查！每次打开都变
call("doc_set_table_cells", {"file_id": FID, "table_id": tid, "cells": [...]})
```
- **身份字段（赛道名称/项目成员/指导老师/参赛单位）不代填**，留给用户；项目名称等技术性字段可填。

### 第 5 步：清理游离段 + 保存 + 校验（同一次执行内）
```python
# 删多余空段：定位「标题 begin」前的空段，删一个验一次结构（防止误删）
hb, _ = find("作品的主要亮点")          # 用下一个标题做右边界
call("doc_delete_paragraph", {"file_id": FID, "idx": hb - 1})
time.sleep(1)                            # 每次删除后重新取结构再判断
# 空段保留 1 个作间距；循环上限 5 次
call("save_file", {"file_id": FID})
# 校验：重新 resolve structure，打印全部节点 + doc_get_images 数图片
```
- `doc_insert_paragraph_with_text` 在「。」后插入可能**把句子断成游离段**（出现只有「。」的孤儿段）——清理时按内容判断，别按固定数量。
- `doc_delete_paragraph` 曾出现"非 JSON 响应但实际没删"——**每删一个都要重新 resolve 校验**，绝不盲删。

## 二次返工的血泪教训（2026-09-27 官方数据集重写时新增）

1. **插图锚点只能用「下一段的起始 begin」**，绝不能用「上一段文本 end+1」——后者落在段落中间会把图注劈成两半，图片粘进图注段，尾巴变成孤儿段。正确姿势：`ins_para(next_para_begin, "")` → 图插进这个空段 → 图注插 `next_index`。
2. **实例跨 Bash 调用 100% 会死**（实测连两次连续的 Bash 调用都撑不住），未 save 的编辑直接丢失。修复类操作必须和首次填充一样：open→全部编辑→save 在 ONE 次脚本内闭环。中途断掉后，先用结构确认磁盘版本的真实状态再动手（缓冲区内容 ≠ 磁盘内容）。
3. **原路径被预览面板占用时 save 会报 "Export file is occupied"**：用 `save_file {"file_id": FID, "file_path": 另一个路径}` 另存（如 `报告书_提交版.docx`），实测有效。save 成功响应也是纯文本（`File saved to: ...`），别当失败。
4. **另存后不要用旧缓冲区继续操作再覆盖回去**：重开 = 从磁盘重载，任何"以为还在"的编辑都没了。每轮修改都必须完整重做整个管线。
5. `doc_find` 对目录页与正文标题的同名文本会返回多个匹配——正文锚点取**最后一个**（最大 begin）。

## 最后交付核对清单
- [ ] 原模板文件未动（比对大小/时间戳）
- [ ] 全文结构干净：无「。」孤儿段、无多余空段（图与下一标题间 ≤1 个）
- [ ] 图片数量正确、全部带居中图注
- [ ] 封面表：已填技术性字段，身份字段留空并提醒用户
- [ ] 章节内容全部有实测出处，正文数字与测试记录一致
- [ ] present_files 打开副本预览

## 通用 docx 编辑坑
本 skill 只覆盖报告填写流程；editor_sdk 的通用坑（并行写导致 offset 漂移、doc_replace_text 只能单段、TOC/outlineLvl、python-docx 兜底、WPS COM 更新域导 PDF 等）见 `docx-local-edit-gotchas` skill，两者配合使用。

## 相关路径（本机实测有效）
- edsdk.py：`D:/Program Files/wb/WorkBuddy/resources/app.asar.unpacked/resources/plugins/workbuddy-builtin/skills/tencent-local-office-edit/edsdk.py`
- 调用方式：`python3 edsdk.py call <method> --json '<payload>'`（cwd 必须是该 skill 目录）
- 模板示例：`D:/Users/Zion/Downloads/13d4ac84-...docx`（封面 5×2 信息表 + 目录 + 五章提纲）
- 工作副本建议放：`agent_app/docs/报告书_工作副本.docx`
