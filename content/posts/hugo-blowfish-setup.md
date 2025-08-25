---
title: "🐡 使用 Hugo + Blowfish 建立現代化部落格"
date: 2025-08-24T10:30:00+08:00
draft: false
description: "分享如何使用 Hugo 靜態網站生成器搭配 Blowfish 主題，打造一個功能完整且美觀的現代化部落格"
tags: ["Hugo", "Blowfish", "靜態網站", "部落格", "教學"]
categories: ["技術分享"]
series: ["部落格建置"]
showToc: true
TocOpen: true
showComments: true
cover:
    image: "posts/hugo-blowfish-setup/featured.svg"
    alt: "Hugo and Blowfish theme setup"
    caption: "建立現代化部落格的完美組合"
---

## 🎯 為什麼選擇 Hugo + Blowfish？

在眾多部落格平台中，我選擇了 Hugo 搭配 Blowfish 主題，主要原因包括：

### Hugo 的優勢
- **⚡ 極快的建置速度**：Hugo 是世界上最快的靜態網站生成器之一
- **🔧 高度可客製化**：豐富的配置選項和模板系統
- **📱 響應式設計**：自動適配各種螢幕尺寸
- **🚀 部署簡單**：可輕鬆部署到各種平台

### Blowfish 主題特色
- **🎨 多種顏色主題**：內建 14+ 種美麗的配色方案
- **🌓 深色模式支援**：自動或手動切換明暗主題
- **🔍 內建搜尋功能**：快速找到所需內容
- **📊 豐富的 Shortcodes**：輕鬆插入圖表、畫廊等元素

## 🛠️ 設置步驟

### 1. 安裝 Hugo

```bash
# macOS (使用 Homebrew)
brew install hugo

# Windows (使用 Chocolatey)
choco install hugo -confirm

# Linux (Ubuntu/Debian)
sudo apt-get install hugo
```

### 2. 建立新網站

```bash
hugo new site my-blog
cd my-blog
```

### 3. 添加 Blowfish 主題

```bash
git submodule add -b main https://github.com/nunocoracao/blowfish.git themes/blowfish
```

### 4. 基本配置

創建 `hugo.toml` 文件：

```toml
baseURL = 'https://yourdomain.com/'
languageCode = 'zh-TW'
title = '你的部落格標題'
theme = "blowfish"

[markup.goldmark.renderer]
  unsafe = true

[taxonomies]
  category = 'categories'
  tag = 'tags'
```

## 🎨 客製化配置

### 選擇配色主題

Blowfish 提供多種內建主題：
- `ocean` - 海洋藍
- `forest` - 森林綠  
- `autumn` - 秋葉橙
- `fire` - 烈焰紅
- `neon` - 霓虹紫

在 `config/_default/params.toml` 中設置：

```toml
colorScheme = "ocean"
defaultAppearance = "light"
autoSwitchAppearance = true
```

### 設定首頁佈局

```toml
[homepage]
layout = "profile"  # 或 "hero", "card", "background"
showRecent = true
showRecentItems = 6
cardView = true
```

### 啟用實用功能

```toml
enableSearch = true
enableCodeCopy = true
showTableOfContents = true
showReadingTime = true
showWordCount = true
```

## 📝 內容創作技巧

### 文章 Front Matter 範例

```yaml
---
title: "文章標題"
date: 2025-08-24
draft: false
description: "文章描述"
tags: ["標籤1", "標籤2"]
categories: ["分類"]
showToc: true
cover:
    image: "path/to/image.jpg"
    alt: "圖片描述"
---
```

### 使用 Shortcodes

Blowfish 提供豐富的 shortcodes：

```markdown
{{< alert >}}
這是一個警告訊息
{{< /alert >}}

{{< button href="https://example.com" >}}
點擊按鈕
{{< /button >}}
```

## 🚀 部署建議

### GitHub Pages
1. 推送程式碼到 GitHub
2. 在 Settings > Pages 中設置部署源
3. 使用 GitHub Actions 自動化部署

### Netlify
1. 連接 GitHub 儲存庫
2. 設置建置命令：`hugo --minify`
3. 發布目錄：`public`

## 💡 優化建議

### 效能優化
- 使用 WebP 格式圖片
- 啟用圖片最佳化
- 配置 CDN

### SEO 優化
- 設置適當的 meta 描述
- 使用結構化數據
- 優化頁面載入速度

### 內容策略
- 定期發布高品質內容
- 建立內容分類系統
- 與讀者互動

## 🎉 結語

Hugo + Blowfish 的組合為我提供了一個功能強大且美觀的部落格平台。透過適當的配置和客製化，你也可以建立一個獨特且專業的網站。

如果你在設置過程中遇到任何問題，歡迎在評論區討論，或者參考 [Blowfish 官方文檔](https://blowfish.page/docs/) 獲取更詳細的資訊。

---

*下一篇文章我將分享如何進一步客製化 Blowfish 主題的樣式和功能。敬請期待！* 🚀