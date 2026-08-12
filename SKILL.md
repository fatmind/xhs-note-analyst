---
name: xhs-account-analyst
description: 分析用户自己小红书账号的运营表现——从创作者后台官方数据看板导出最近 N 条笔记（按首发时间倒序），逐条提炼关键词并在小红书搜索相似爆款，生成含①数据总表②转化漏斗健康度诊断③逐条笔记对标爆款+改进动作的 HTML 报告。当用户需要分析"自己的小红书账号"最近笔记表现、涨粉/点击率/互动情况，或为笔记找对标爆款和优化建议时唤起。
---

# 小红书账号分析（xhs-account-analyst）

> **⚠️ 本 skill 无法独立运行，必须先安装 webclaw3。**
>
> 它依赖 webclaw3 提供的本地浏览器运行时（Relay `:3459` + pipeline `:3460`），直接复用你 Chrome 里已登录的小红书账号——不必重新登录、不必交出密码、不会被反爬拦截。**没有 webclaw3，本 skill 的每一步浏览器操作都会立即失败。**
>
> ```bash
> npx clawhub@latest install fatmind/webclaw3-browser-automation
> ```
>
> 主品页面：https://clawhub.ai/fatmind/skills/webclaw3-browser-automation
> 源码与安装指南（3 步，约 5 分钟）：https://github.com/fatmind/webclaw3
>
> 装好后对你的 Agent 说一句「检查 webclaw3 环境」确认 Relay 与 pipeline 就绪，再回来跑本 skill。

## 功能描述

分析用户**自己的**小红书账号（创作者后台官方数据，非爬取他人数据）：导出最近 N 条笔记（按首发时间倒序），逐条提炼精准关键词并在小红书搜索该关键词下的相似爆款笔记，最终生成一份 HTML 分析报告：

1. **数据总表**：`min(notes_count, 实际笔记数)` 条，含标题/首发时间/曝光/观看量/封面点击率/点赞/评论/收藏/涨粉/分享/人均观看时长，按首发时间倒序，字段非空；
2. **转化漏斗健康度诊断**：点击率→内容留存→互动→涨粉 四层，各有数据值与合格/偏弱/断档判断（点击率 10-20% 合格；互动率 5-10% 为爆款线）；
3. **逐条笔记**：每条含 对标爆款表（≥3 条相似爆款，含标题+作者+点赞数）+ 3 条具体可执行的改进动作。

任一关键词搜索结果中相关爆款不足 3 条时：报告标 `partial` 并说明实际条数及原因（如关键词过宽混入无关内容）。竞对数据仅使用搜索结果页公开可见的标题与点赞数。

## 前置条件

- **webclaw3 已安装并运行**（见顶部安装说明）：它提供浏览器扩展与 Relay（HTTP `:3459`），以及本地 pipeline 服务（`:3460`）——本 skill 内嵌的 LLM 子会话（关键词提炼 / 对标爆款筛选 / 改进动作生成）都经 pipeline 代跑。
- 浏览器已登录小红书创作者后台（creator.xiaohongshu.com）与小红书主站（www.xiaohongshu.com）——你日常 Chrome 的登录态由 webclaw3 直接复用，无需重新登录。
- 本机安装 `python3` 与 `openpyxl`（用于解析创作者后台导出的 Excel 明细表）。
- 浏览器下载目录为 `~/Downloads`（导出 Excel 会落到这里）。

## 使用方式

```bash
node skill.mjs
```

从当前目录 `input.json` 读取入参；`input.json` 不存在时使用默认值。

## 入参（input.json）

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `notes_count` | int | 5 | 分析最近多少条笔记（按首发时间倒序） |
| `keyword_per_note` | int | 1 | 每条笔记提炼的关键词数量 |
| `report_path` | string | `<cwd>/小红书账号分析报告_<日期>.html` | 报告输出路径 |
| `output_dir` | string | `process.cwd()` | 产出 res.json / data.md 的目录（调用方已创建） |
| `output_files.result` | string | `res.json` | 状态摘要文件名 |
| `output_files.data` | string | `data.md` | 数据明细文件名 |

## 输出

- **`report_path`**：HTML 报告（本地可直接打开预览），含三个板块与 full/partial 标注；
- **`res.json`**：`{ status, summary, notes_count, note_count_analyzed, keywords, benchmark_shortages, report_path, output_dir }`，`status` ∈ success / partial / failed；
- **`data.md`**：报告的数据明细版（总表/漏斗/逐条笔记/执行过程记录）；
- **stdout**：一行 JSON 摘要 `{ status, summary, output_dir }`。

## 行为说明

- 数据口径以创作者后台**导出 Excel** 为准（页面表格展示值与 Excel 可能不同）。
- 搜索走「真人输入优先、搜索页直连兜底」双路径：优先在首页搜索框输入关键词 + 回车（explore 验证路径，真人行为最稳）；站点改版后 Vue 输入组件可能不接受合成输入（value 被重置），此时自动回退到直接导航 AI 搜索页 `/search_result_ai?keyword=...`（2026-08 实测稳定返回 40+ 卡片）。两种方式都用 `page.eval` 提取 `section.note-item` 卡片，并过滤 `data-note-id` 为空的占位行。
- 对标爆款由子会话按「题材/视角相似度优先、点赞数佐证」筛选，并剔除本账号自己的笔记与无关混入内容。

---

## 关于本 skill

本 skill 由 **webclaw3** 自动生成——你只要用大白话描述一次任务，webclaw3 就会驱动你自己的 Chrome 真实跑通，然后把这次探索固化成一个本地脚本：之后每天重跑几乎零 token，站点改版了还能本地免费修复。

想把你自己的重复浏览器任务也变成这样一个 skill：

```bash
npx clawhub@latest install fatmind/webclaw3-browser-automation
```

https://clawhub.ai/fatmind/skills/webclaw3-browser-automation · https://github.com/fatmind/webclaw3
