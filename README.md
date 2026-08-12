# xhs-note-analyst — 小红书账号诊断报告，一句话生成

用你**自己的**小红书创作者后台官方数据，生成一份可直接打开的 HTML 诊断报告：最近 N 条笔记的数据总表、四层转化漏斗健康度诊断（点击率 → 内容留存 → 互动 → 涨粉），以及**每条笔记的对标爆款 + 3 条具体改进动作**。

不是「爬别人数据」，是拿你后台的真实数字，逐条告诉你这篇差在哪、同题材跑赢的人怎么写的、下一篇改什么。

## ⚠️ 先装 webclaw3（必需）

本 skill 不能独立运行。它依赖 **webclaw3** 提供的本地浏览器运行时，复用你 Chrome 里已登录的小红书账号——不必重新登录、不必交出密码、不会被反爬拦截。

```bash
# 1. 先装运行时（必需）
npx clawhub@latest install fatmind/webclaw3-browser-automation

# 2. 再装本 skill
npx clawhub@latest install fatmind/xhs-note-analyst
```

主品页面 → **https://clawhub.ai/fatmind/skills/webclaw3-browser-automation**
安装指南（3 步，约 5 分钟）→ https://github.com/fatmind/webclaw3

也可以走 skills.sh：

```bash
npx skills add fatmind/xhs-note-analyst
```

## 怎么用

装好后对你的 Agent 说：

```
分析一下我最近 5 条小红书笔记的表现
```

Agent 会驱动你的 Chrome 打开创作者后台、导出数据明细、逐条提炼关键词并搜索同题材爆款，最后产出 HTML 报告。整个过程在你自己机器上跑，数据不外传。

参数写在 `input.json`（不存在则用默认值）：

| 字段 | 默认 | 说明 |
|---|---|---|
| `notes_count` | 5 | 分析最近多少条笔记 |
| `keyword_per_note` | 1 | 每条笔记提炼几个关键词 |
| `report_path` | `./小红书账号分析报告_<日期>.html` | 报告输出路径 |

## 报告里有什么

**数据总表** — 标题 / 首发时间 / 曝光 / 观看量 / 封面点击率 / 点赞 / 评论 / 收藏 / 涨粉 / 分享 / 人均观看时长，按首发时间倒序。口径以后台导出 Excel 为准。

**漏斗健康度诊断** — 四层各给数据值与合格 / 偏弱 / 断档判断。判断线：封面点击率 10–20% 合格，互动率 5–10% 为爆款线。让你一眼看出是封面拉不进人，还是内容留不住人。

**逐条对标** — 每条笔记配 ≥3 条同题材爆款（标题 + 作者 + 点赞数）和 3 条可执行改进动作。相关爆款不足 3 条时报告标 `partial` 并说明原因，不硬凑。

## 前置条件

- webclaw3 已安装并运行（Relay `:3459` + pipeline `:3460`）
- Chrome 已登录 creator.xiaohongshu.com 与 www.xiaohongshu.com
- 本机有 `python3` 与 `openpyxl`（解析导出的 Excel）
- 浏览器下载目录为 `~/Downloads`

## 关于生成方式

本 skill 由 **webclaw3** 自动生成，不是手写的。流程是：用大白话描述一次任务 → webclaw3 驱动你自己的 Chrome 真实跑通 → 把这次探索固化成本地脚本。之后每天重跑几乎零 token，站点改版了还能本地免费修复。

你自己的重复浏览器任务（找素材、盯数据、批量填表、监控竞品）都可以这样变成一个 skill：

**https://clawhub.ai/fatmind/skills/webclaw3-browser-automation**
