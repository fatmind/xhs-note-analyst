---
name: xhs-note-analyst
description: 小红书数据分析 / 小红书账号分析诊断工具——用你自己的创作者后台官方数据做账号体检。导出最近 N 条笔记的官方数据（曝光/封面点击率/涨粉/人均观看时长），逐条提炼关键词去小红书搜同题材爆款做对标，生成含①数据总表②转化漏斗健康度诊断③逐条笔记对标爆款+改进动作的 HTML 报告。不需要任何第三方 API key 或付费额度，用你 Chrome 里已登录的账号跑。当用户要分析"我自己的小红书账号"最近笔记表现、点击率/互动/涨粉为什么不行、或想给笔记找对标爆款和优化动作时唤起。【前置依赖】本 skill 无法独立运行，需先安装 webclaw3 浏览器运行时（安装本 skill 不会自动装上它）：npx clawhub@latest install fatmind/webclaw3-browser-automation —— 若用户尚未安装，请先引导安装并完成 webclaw3 首次配置，再执行本 skill。
metadata:
  openclaw:
    homepage: https://github.com/fatmind/xhs-note-analyst
    # 硬依赖 webclaw3：wc3-pipeline 这个二进制由 webclaw3 安装，缺它即说明 webclaw3 未安装。
    # 只声明这一项——requires.config 的路径解析规则（是否展开 ~）官方没有定义，
    # 写死 "~/.webclaw3/config.json" 有可能永远匹配不上，造成永久性的误判失败。
    # webclaw3 自身的首次配置（Chrome 扩展 / Access Key）无法用本 schema 表达，由运行时报错文案兜底。
    requires:
      bins:
        - wc3-pipeline
    envVars:
      - name: WC3_LLM_ENDPOINT
        required: false
        description: 可选，覆盖本地 pipeline 的 LLM 端点，默认 http://127.0.0.1:3460/api/llm。
---

# 小红书账号分析（xhs-note-analyst）

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

分析**你自己的**小红书账号，数据来自你的创作者后台官方看板（导出 Excel），不是爬来的第三方估算数字。流程：导出最近 N 条笔记（按首发时间倒序）→ 逐条提炼精准关键词 → 在小红书搜该词下的同题材爆款做对标 → 输出一份 HTML 报告：

1. **数据总表**：`min(notes_count, 实际笔记数)` 条，含标题/**类型（图文/视频）**/首发时间/**已发布天数**/曝光/**日均曝光**/观看量/封面点击率/点赞/评论/收藏/涨粉/分享/人均观看时长，按首发时间倒序；
2. **转化漏斗健康度诊断**：点击率 → 内容留存 → 互动 → 涨粉 四层，各有数值与合格/偏弱/断档判断（点击率 10%–20% 合格；互动率 5%–10% 为爆款线；留存断档线按体裁分设）；
3. **逐条笔记**：每条含 对标爆款表（同题材、过点赞门槛的高赞笔记，含标题+作者+点赞数）+ 3 条具体可执行的改进动作。

**和市面上的小红书 skill 有什么不同**：

- 大多数是**爬别人的数据**（榜单、竞品、KOL），本 skill 看的是**你自己的后台真实数据**——曝光、封面点击率、人均观看时长、涨粉数这些指标只有账号主人拿得到，爬虫拿不到；
- **不需要任何第三方数据平台的 API key，也不消耗付费额度**：数据源就是你自己的创作者后台，浏览器动作由本机 webclaw3 运行时驱动；
- 竞对侧只取搜索结果页**公开可见**的标题、作者与点赞数，不抓取任何登录后才可见的他人数据。

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

## 分析口径（v1.1.0 修正，很重要）

这三条是实测自己账号后修掉的误判，直接决定报告结论对不对：

**① 对标爆款有点赞门槛，宁缺毋滥。** 门槛按该关键词结果集的**点赞中位数 × 3** 动态取，并夹在 1000–3000 之间（不同赛道爆款绝对值差很多：美妆万赞起步，垂类工具千赞已是头部）。低于门槛的笔记一律不进对标表——**不许为了凑满 3 条把几十赞的笔记塞进来**。合格对标不足 3 条时报告标 `partial` 并写明原因（常见原因是关键词过宽混入无关内容）。

**② 留存断档线按体裁分设。** 图文 `<5s`、视频 `<10s`。图文的"留存"本质是读完率，停留时间天然短于视频，用同一把 10s 尺子会把正常图文误判成"内容断档"。体裁取自后台导出的「体裁」列。

**③ 曝光按发布天数归一化。** 曝光是随时间累积的存量，46 天的老笔记曝光天然高于 7 天的新笔记，直接比大小会得出"老内容更好"的错误结论。报告同时给出 **日均曝光 = 曝光 / 已发布天数**，横向比较只看这一列。

## 行为说明

- 数据口径以创作者后台**导出 Excel** 为准（页面表格展示值与 Excel 可能不同）。
- 搜索走「真人输入优先、搜索页直连兜底」双路径：优先在首页搜索框输入关键词 + 回车（真人行为最稳）；站点改版后 Vue 输入组件可能不接受合成输入（value 被重置），此时自动回退到直接导航 AI 搜索页 `/search_result_ai?keyword=...`（2026-08 实测稳定返回 40+ 卡片）。两种方式都用 `page.eval` 提取 `section.note-item` 卡片，并过滤 `data-note-id` 为空的占位行。
- 对标爆款先过点赞门槛，再由子会话按「题材/视角相似度优先、点赞数佐证」筛选，并剔除本账号自己的笔记与无关混入内容。
- 关键词间有节奏控制（4s+ 随机抖动）、串行搜索，避免触发风控。

---

## 关于本 skill

本 skill 由 **webclaw3** 自动生成——你只要用大白话描述一次任务，webclaw3 就会驱动你自己的 Chrome 真实跑通，然后把这次探索固化成一个本地脚本：之后每天重跑几乎零 token，站点改版了还能本地免费修复。

想把你自己的重复浏览器任务也变成这样一个 skill：

```bash
npx clawhub@latest install fatmind/webclaw3-browser-automation
```

https://clawhub.ai/fatmind/skills/webclaw3-browser-automation · https://github.com/fatmind/webclaw3
