---
name: feishu-summary-sync
description: Summarize web links or provided text into local Markdown notes, then sync them to Feishu Drive as docx documents and an annual index sheet. Use when handling信息概要汇总、URL摘要落盘、飞书文档同步、年度索引表维护、Feishu docx/sheet导入、或修复 lark-cli 在 Windows 下的参数与编码问题。
---

# Feishu Summary Sync

将“信息概要汇总”任务稳定落地为：
- 本地 `Markdown` 笔记
- 飞书 `docx` 文档
- 飞书年度索引 `sheet`

优先保持流程稳定，不要猜测 `lark-cli` 参数格式。

## 核心流程

1. 先提取用户提供的 `URL`
2. 逐个读取内容并整理为中文 `Markdown`
3. 本地落盘为 `YYYY-MM-DD-概要标题.md`
4. 默认同步飞书：
   - 正文文档走 `docx`
   - 年度索引优先走 `sheet`
5. 最后回报本地路径、飞书目录链接、飞书文档链接

## 本地文件命名

- 正文笔记：`YYYY-MM-DD-概要标题.md`

## 飞书目录规范

- 根目录：`信息概要笔记`
- 月目录：`YYYY-MM`
- 日目录：`YYYY-MM-DD`
- 年度索引表：`信息概要汇总索引-YYYY`

## 必须记住的坑

### 1. `lark-cli sheets +write/+append` 不要作为首选封装

现象：
- `sheets +write/+append` 容易返回 `90204 valueRange is wrong`
- 这层封装在 Windows 下不稳定，容易把 `values` 或 `range` 传错

结论：
- **不要把 `sheets +write/+append` 当成默认写表方案**
- 优先使用通用 API 直写：
  - 单范围：`PUT https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/<token>/values`
  - 批量：`POST https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/<token>/values_batch_update`

可用命令示例：

```bash
python - <<'PY'
from pathlib import Path
Path('.cache').mkdir(exist_ok=True)
Path('.cache/sheets-write-data.json').write_text(
    '{"valueRange":{"range":"0IRGkM!A8:B8","values":[["Hello",1]]}}',
    encoding='utf-8'
)
Path('.cache/sheets-batch-data.json').write_text(
    '{"valueRanges":[{"range":"0IRGkM!A9:B10","values":[["Batch1",1],["Batch2",2]]}]}',
    encoding='utf-8'
)
PY

lark-cli api PUT https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/<token>/values --data @.cache/sheets-write-data.json
lark-cli api POST https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/<token>/values_batch_update --data @.cache/sheets-batch-data.json
```

### 2. `drive files patch` 和 `Sheets API` 的 JSON 文件必须是 UTF-8 无 BOM

现象：
- `--params invalid format, expected JSON object`

原因：
- 传给 `--params @file` / `--data @file` 的 JSON 若带 `BOM`，`lark-cli` 在 Windows 下容易解析失败
- 这会同时影响 `drive files patch` 和 `Sheets API` 直写请求

正确做法：
- 用无 `BOM` 的 UTF-8 写 JSON 文件
- 再调用：
  - `lark-cli drive files patch --params @params.json --data @data.json --yes`
  - `lark-cli api PUT https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/<token>/values --data @data.json`

跨平台示例：

```bash
python - <<'PY'
from pathlib import Path
Path('params.json').write_text('{"file_token":"...","type":"docx"}', encoding='utf-8')
Path('data.json').write_text('{"new_title":"..."}', encoding='utf-8')
PY

lark-cli drive files patch --params @params.json --data @data.json --yes
```

## 推荐工作流

### 正文文档

- 本地 `.md` 生成完成后，直接：
  - `lark-cli drive +import --file <md> --folder-token <day-folder> --type docx --name <title>`

注意：
- 若 `--name` 含空格或 shell 易拆词，可先使用简化名称导入
- 导入完成后，再用 `drive files patch` 回写成最终标题

### 年度索引表

默认走 **直写飞书 sheet**。

推荐流程：

1. 先用 `lark-cli sheets +info --spreadsheet-token <token>` 获取 `sheet_id`
2. 用 `Sheets API` 直写：
   - `PUT /open-apis/sheets/v2/spreadsheets/<token>/values`
   - `POST /open-apis/sheets/v2/spreadsheets/<token>/values_batch_update`
3. `range` 使用 `sheetId!A1:B2` 这种格式
4. 写入前确保 JSON 是 **UTF-8 无 BOM**
5. 写入后再次用 `lark-cli sheets +info` / `+read` 校验

## 验证清单

交付前至少确认：

- 本地 `.md` 文件存在
- 飞书 `docx` 文档链接可用
- 飞书索引表存在
- 索引表工作表名为当月 `YYYYMM`
- 索引内容已写入
- 飞书文档标题与本地文件名一致

## 输出注意点

- 本地文件未生成前，不要宣称完成
- 飞书未同步完成前，不要宣称“飞书文档已经创建/更新完成”
- 如果只完成一部分，要明确标注：
  - 本地成功
  - 飞书正文成功
  - 索引表失败 / 部分成功
