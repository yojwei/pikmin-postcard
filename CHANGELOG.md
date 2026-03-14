# 更新紀錄

本檔案依 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/) 格式撰寫。

---

## [Unreleased]

### Added

- **照片上傳雙入口**：在「選擇照片或拍照」區塊新增兩個獨立按鈕——「開啟相機」（`capture="environment"`）與「開啟相簿」（一般 file input），行動裝置可直接拍照或從相簿挑選
- **分享功能**：新增「分享明信片」按鈕，呼叫 Web Share API（`navigator.share`）分享 PNG 檔案；裝置或瀏覽器不支援時自動回退為下載
- **郵票變體系統**：右側裝飾面板加入 10 種郵票樣式（ruby-blossom、mint-splash、citrus-pop、amethyst-dream、cobalt-night、sakura-mist、aurora-sky、sunlit-leaf、berry-garden、moonstone-frost），每種樣式具獨立邊框、主題圖案（SVG）與隨機面額（30–90，步進 5）；每次頁面載入隨機選取

### Changed

- **照片渲染方式**：左側照片區改為以 CSS `background-image` + `background-size: cover` 的 `<div>` 呈現（原為 `<img>`），解決 html2canvas 匯出時 `object-fit: cover` 造成的圖片壓縮問題
- **明信片比例**：行動版（預設）調整為 `aspect-[1.45/1]`，`sm` 斷點以上維持 `aspect-[1.6/1]`，改善小螢幕可讀性
- **皮克敏圖片來源格式**：`pikmin_sources.json` 由舊版 `sourceGroups`（群組陣列）重構為 `colorGroups`（每色對應 URL 清單），並保留舊格式自動轉換的向下相容邏輯
- **皮克敏最大渲染寬度**：上限由原值調整為 **40%**（縮放後），確保多隻皮克敏時不超出照片邊界
- **控制面板排版**：步驟區塊改用圓形編號標記（粉色圓圈 1/2/3），強化視覺流程引導；所有輸入元件與按鈕統一使用圓角卡片風格

### Fixed

- **下載/分享 CTA 停用狀態**：上傳照片前 `disabled` 屬性正確套用，防止在無照片時觸發匯出
- **按鈕鍵盤無障礙**：`<label>` 類按鈕（相機/相簿）補上 `role="button"`、`tabindex="0"` 及 `keydown.enter`/`keydown.space` 事件處理，確保鍵盤可操作
- **焦點樣式**：所有互動元件補上 `focus-visible:ring-*` 樣式，解決鍵盤操作時焦點不可見的問題
- **ARIA 語意**：定位按鈕加入 `aria-busy`；皮克敏選取按鈕加入 `aria-pressed`；裝飾性 SVG 標記 `aria-hidden="true" focusable="false"`；定位提示文字以 `aria-describedby` 關聯按鈕
- **觸控目標尺寸**：主要按鈕統一設定 `min-h-[48px]`（部分加上 `min-w-[48px]`），符合行動裝置建議觸控面積
- **匯出穩定性**：匯出前等待 `document.fonts.ready` 與所有 `<img>` 的 `decode()` 完成，再等候兩個 `requestAnimationFrame`，減少字型或圖片尚未繪製即截圖的問題
