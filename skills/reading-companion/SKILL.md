---
name: reading-companion
description: >-
  讀論文或 repo 項目，產出結構化批判筆記，按用戶要求存入 Obsidian vault。Trigger:
  用戶要求讀、總結、評析 arxiv 論文、GitHub 項目或技術文章，或單獨貼出此類 URL。
---

# Reading Companion

讀技術內容，產出帶批判視角的結構化筆記。約定體例與存放位置；讀法由模型自行判斷。

## 原則

- 內容已在上下文（貼文、上傳、先前抓取）則直接用，不重複抓取。
- 用戶提供的原文 > 官方源 > 鏡像 > 二手轉述；非一手來源標 `置信度: medium/low`，注明實際來源。
- 事實與推斷分開；信息缺失就說缺失，不用合理猜測填補。
- 不憑 README 空談 benchmark 或代碼理解；引用結果須帶 metric、benchmark、baseline、delta。

## 抓取

WebFetch 優先；被阻擋時退到 `~/Scripts/fetch-url <url>`（本地網絡 + readability，24h 緩存）。仍失敗則停止並向用戶要原文，不硬湊。

## 筆記體例

```markdown
# {標題}

- **来源**: {URL / uploaded file / pasted text}
- **日期**: {發表日期}
- **作者/机构**: {…}
- **类型**: paper | repo | article | local
- **置信度**: high | medium | low

## 核心问题
{1-3 句：這項工作面對的缺口}

## 核心方案
{編號列表：機制名 + 1-2 句解釋}

## 证据
{metric + benchmark + baseline + delta；repo 則為維護信號。無定量證據則省略}

## 风险与弱点
{⚠️ 前綴列點：隱藏假設、未承認的局限、工程風險}

## 待验证问题
{以問句提出，不下斷語}
```

簡要模式（用戶只要概覽）：核心问题 + 核心方案 + 一句風險，其余省略。

## 語言

中文行文，技術術語、方法名、benchmark 名保留英文原文；代碼與公式原樣。

## 存檔

僅在用戶要求時保存。路徑：`<obsidian-vault>/reading-notes/{YYYY-MM-DD}_{slug}.md`，保存後回報路徑。
