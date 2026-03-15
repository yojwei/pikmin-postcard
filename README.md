# Pikmin Bloom 明信片產生器

一個純前端網頁應用，用於製作 Pikmin Bloom 風格的明信片。上傳照片、加入標題與地點、撒上幾隻皮克敏，然後匯出 PNG。

---

## 功能簡介

產生明信片風格圖片，包含：

- 左側使用者上傳的照片（約佔卡片 65%）
- 右側裝飾面板（約 35%），含水彩花瓣圖案、「PIKMIN BLOOM」標籤與隨機郵票圖案
- 以隨機位置、大小與旋轉角度疊加於卡片上的皮克敏角色
- 疊加在照片上的標題與地點文字

---

## 特色功能

### 照片上傳

照片控制區提供兩個獨立入口：

- **開啟相機**：使用 `<input capture="environment">`，在行動裝置上直接開啟後鏡頭拍照
- **開啟相簿**：使用一般 `<input type="file">`，從裝置相簿或檔案系統選取圖片

兩個按鈕均以 `<label role="button" tabindex="0">` 實作，支援鍵盤 Enter/Space 觸發，視覺上具有一致的粉色互動樣式。

### 預覽與下載 CTA

- 尚未上傳照片時，預覽區左側顯示**空白提示佔位符**，引導使用者執行步驟 3
- **下載明信片**與**分享明信片**按鈕在上傳照片前保持**停用狀態**（`disabled`），避免誤觸
- 匯出處理中時按鈕切換為 spinner + 文字「處理中…」，並互相鎖定防止同時執行

### 標題與地點

- 自訂標題，或點擊✨按鈕從預設繁體中文詞彙庫中隨機產生
- 手動輸入地點，或透過瀏覽器定位 + OpenStreetMap 反向地理編碼（Nominatim）自動填入
- 開啟頁面時會自動嘗試定位；成功解析後會更新地點欄位
- 已解析的地點會儲存在 `localStorage`，下次開啟頁面時優先重用
- 定位中顯示 spinner 並以 `aria-busy` 標記，完成後自動填入地點欄位

### 皮克敏

- 選擇要包含的 7 種顏色：紅、黃、藍、白、紫、石、翼
- 若未選擇任何顏色，則 7 種顏色皆為候選
- **隨機配置** 會產生 1–6 隻皮克敏，具有以下特性：
  - 大小與數量成反比縮放，最終渲染寬度上限為 **40%**（縮放後）
  - 分布於三個區域：左緣、右緣或底部
  - 每個實例執行 60 次候選評分迴圈，以減少重疊並最大化可見性
  - 旋轉角度（依區域 ±14–20°）、輕微縮放抖動（0.88–1.04×；單隻時 0.95–1.08×）、並依垂直位置決定 z-index 層級

### 郵票

右側裝飾面板內嵌一枚隨機郵票，共 10 種樣式（ruby-blossom、mint-splash、citrus-pop 等），每種樣式有獨立的邊框顏色、主題圖案（寶石、花朵、星星、彗星…）與隨機面額（30–90，步進 5）。

### 圖片來源

皮克敏圖片從 `pikmin_sources.json` 載入，該檔案將每種顏色對應至 URL 清單：

```json
{
  "colorGroups": {
    "red":    ["https://...", "https://..."],
    "blue":   ["..."],
    ...
  }
}
```

- 每次載入時以 `cache: 'no-store'` 方式取得該檔案
- 來源群組索引儲存於 `localStorage`，每個 session 輪換
- 舊版 `sourceGroups`/`groups` 格式會自動轉換為目前的 `colorGroups` 格式

**備援鏈**（每個皮克敏實例，於圖片載入失敗時）：

1. 該顏色來源清單中的下一個 URL
2. 硬編碼的 Wikia URL（Pikmin 4 wiki）
3. 帶有顏色 emoji 佔位符的內嵌 SVG

### 匯出與分享

- **下載明信片**：將明信片匯出為 PNG，儲存為 `pikmin-postcard-{timestamp}.png`
- **分享明信片**：呼叫 Web Share API（`navigator.share`）；不支援時自動回退為下載
- 匯出前等待字型與圖片穩定（`document.fonts.ready` + `img.decode()`），再以 `html2canvas 1.4.1`（2× 比例、`useCORS: true`）渲染

---

## 無障礙與響應式

- 所有按鈕與互動元件均設有 **`focus-visible` ring** 樣式，支援鍵盤操作
- 觸控目標最小高度 **48px**（`min-h-[48px]`），符合行動裝置可點擊範圍建議
- 使用 `aria-label`、`aria-pressed`、`aria-busy`、`aria-describedby` 等屬性提升輔助工具相容性
- 裝飾性 SVG 圖示標記 `aria-hidden="true" focusable="false"`，避免輔助工具朗讀
- **明信片比例**：行動版為 `aspect-[1.45/1]`，`sm` 斷點以上為 `aspect-[1.6/1]`
- 相機/相簿選擇按鈕在手機採單欄，`sm` 以上自動切換為雙欄並排

---

## 執行方式

無需建置步驟或伺服器。直接在現代瀏覽器中開啟 `index.html`：
所有相依套件均於執行時從 CDN 載入：

- **Vue 3** — 響應式 UI
- **Tailwind CSS** — 樣式
- **html2canvas 1.4.1** — DOM 轉 PNG 匯出

> 若要讓 GPS 與跨來源圖片功能正常運作，部分瀏覽器需要透過 HTTP 提供頁面，而非以本機 `file://` URL 開啟。使用簡易本機伺服器（例如 `npx serve .`）可解決此問題。

---

## 檔案說明

| 檔案                  | 用途                                         |
| --------------------- | -------------------------------------------- |
| `index.html`          | 完整應用程式（HTML + Vue 3 + CSS）           |
| `pikmin_sources.json` | 依顏色分類的皮克敏圖片 URL                   |
| `pikmin_images.md`    | 依顏色與裝飾類型整理的皮克敏圖片資源參考清單 |

---

## 重要行為說明

- **未選擇顏色** → 7 種皮克敏顏色皆可隨機選取
- **下載/分享停用條件**：`!photo || isDownloading || isSharing`，三個條件任一為真時按鈕不可點擊
- **空白預覽佔位符**：`v-if="!photo"` 時左側照片區顯示提示框，引導使用者上傳照片
- **照片渲染方式**：以 CSS `background-image` + `background-size: cover` 的 `<div>` 呈現（而非 `<img>`），使 html2canvas 匯出時保有正確裁切比例
- **地點初始化**：頁面載入時會自動嘗試定位；若先前已有已解析地點，會先從 `localStorage` 帶入並重用
- **定位備援鏈**：village → suburb → neighbourhood → town → city_district → county → city（取 Nominatim 回應中第一個有值的欄位）
- **圖片載入追蹤**：每個皮克敏實例追蹤 `attemptedSourceIndices` 以避免重試失敗的 URL
- **載入控管可見性**：皮克敏圖片在 `load` 事件成功前保持隱藏；重試期間再次隱藏，直到載入成功；所有來源均失敗後才顯示 emoji 備援
- **匯出檔名**：`pikmin-postcard-{Date.now()}.png`
- **文字渲染**：標題與地點使用 Tailwind 響應式大小類別（斷點步進）；右側面板裝飾文字使用 CSS `clamp()`；text-shadow 確保在照片上的文字可讀性
