# 拍照模式整合

> 文件版本：v3  
> 狀態：規劃中（部分已實作）  
> 語言：Traditional Chinese

---

## 1. 背景與目前狀態

### 1.1 專案背景
本專案為皮克敏風格明信片產生器（`index.html`），使用 Vue 3（CDN）+ Tailwind CSS，
讓使用者拍照或選圖、加入皮克敏疊加層與文字後匯出明信片。

### 1.2 目前已實作（index.html 現況）

| 功能 | 狀態 | 對應程式碼 |
|------|------|-----------|
| 相簿選圖 | ✅ 已實作 | `<input type="file" id="galleryInput">` |
| 原生相機（file input capture） | ✅ 已實作 | `<input type="file" capture="environment" id="cameraInput">` |
| Pseudo AR 相機（getUserMedia） | ✅ 已實作 | `startPseudoCamera()`、`pseudoCameraVideoRef`、`pseudoCameraStream` |
| 拍照匯入（canvas 截圖） | ✅ 已實作 | `capturePhotoFromPseudoCamera()`：drawImage → toDataURL PNG |
| 停止相機 | ✅ 已實作 | `stopPseudoCamera()`：track.stop() + pause + srcObject = null |
| 模式選擇（auto / pseudo-ar / file） | ✅ 已實作 | `preferredCaptureMode`（ref）→ `resolvedCaptureMode`（computed） |
| 皮克敏疊加層 | ✅ 已實作 | `selectedPikminItems` / `getPikminOverlayStyle()` |
| 文字疊加（標題 + 地點） | ✅ 已實作 | 絕對定位於照片區左上角，含 text-shadow |
| 全螢幕相機模式 | ❌ 未實作 | — |
| 橫式方向鎖定 | ❌ 未實作 | — |
| 相機畫面縮放（pinch-to-zoom） | ❌ 未實作 | — |
| 重拍流程（拍後可重拍） | ⚠️ 部分：拍後 `photo` 被設定，需手動再次「開始相機」 |
| **AR 相機預覽**（live 畫面疊加明信片 AR 層） | ❌ 未實作 | 核心新需求：§3.4 詳述 |
| WebXR AR 模式 | ❌ 永遠停用（`WEBXR_FEATURE_ENABLED = false`） | — |

### 1.3 照片區 UI 結構（目前）

```
明信片本體（aspect-ratio 1.45/1 ~ 1.6/1）
└── 左側照片區（寬 65%，height 100%）
    ├── <video> pseudoCameraVideoRef（isPseudoCameraLive 時）
    ├── <div> 背景圖（photo 已選時）
    ├── <div> 漸層遮罩（上深下淡）
    ├── 文字疊加：標題 + 地點（絕對定位，左上角）
    └── 皮克敏疊加層（絕對定位，z-[8]）
```

### 1.4 本次計畫要解決的問題

1. 「開啟相機」按鈕目前呼叫原生 file input（`capture="environment"`），與 Pseudo AR 模式並存但 UI 脈絡不清。
2. Pseudo AR 相機目前嵌於明信片卡片內（65% 寬），取景框架太小且無全螢幕選項。
3. **核心新需求**：開啟相機時，希望將明信片 AR 元素（明信片導引框、漸層遮罩、標題 / 地點預覽、皮克敏疊加）即時呈現於相機 live 畫面上，讓使用者構圖時即可預覽最終明信片效果。
4. 拍照後需重新點「開始相機」才能重拍，流程不直覺。
5. 無縮放功能，難以精確對焦主體。

---

## 2. 術語定義

| 術語 | 定義 | 備註 |
|------|------|------|
| **原生相機（Native Capture）** | `<input type="file" capture="environment">` 觸發系統相機 App | 現有「開啟相機」按鈕的實作方式 |
| **Pseudo AR 相機** | 以 `getUserMedia` 取得後鏡頭串流，嵌入頁面 `<video>` 元素，截圖後轉為明信片底圖 | 已實作；`resolvedCaptureMode === 'pseudo-ar'` |
| **AR 相機預覽（AR Camera Preview）** | Pseudo AR 相機開啟後，在 live 相機畫面上即時疊加明信片 AR 元素（導引框、漸層遮罩、標題 / 地點文字、皮克敏），讓使用者在拍照前即可預覽最終明信片效果 | ❌ 待實作；為本次計畫核心功能 |
| **AR 疊加層可見性** | AR 元素僅在 live 預覽期間顯示；截圖時拍下的是純鏡頭原始畫面（不含 AR 元素），AR 元素由明信片 UI 另外疊加 | 詳見 §3.4 |
| **AR 預覽開關（AR Toggle）** | 全螢幕相機 overlay 內的按鈕，讓使用者切換「顯示 / 隱藏 AR 疊加層」 | 隱藏時僅顯示原始鏡頭畫面 |
| **全螢幕相機模式** | 開啟 Pseudo AR 相機時，以全視窗 overlay 蓋過頁面，提供更大取景框並承載 AR 預覽 | ❌ 待實作 |
| **橫式鎖定** | 相機全螢幕時嘗試呼叫 `screen.orientation.lock('landscape')` | ❌ 待實作；iOS 不支援，需降級 |
| **重拍** | 拍照後未離開相機流程，可立即丟棄目前截圖並回到 live 預覽 | ⚠️ 目前需手動重新點「開始相機」 |
| **pinch-to-zoom** | 雙指縮放手勢，透過 `videoConstraints.zoom` 或 CSS transform 實現相機縮放 | ❌ 待實作 |

---

## 3. 功能規格

### 3.1 相簿入口

**目前行為**：`<label for="galleryInput">` → `<input type="file" accept="image/*">` 開啟系統圖庫。  
**計畫調整**：
- 保留現有「開啟相簿」按鈕，**不刪除**。
- 在 `resolvedCaptureMode === 'file'` 模式下，「開啟相機」按鈕改為觸發 `openFilePicker()`（即呼叫 `cameraInputRef.value.click()`）。
- 無其他預期 UI 變更。

### 3.2 相機開啟流程

```
使用者點擊「開啟相機」
  │
  ├─ resolvedCaptureMode === 'file'
  │     └─ 呼叫 cameraInputRef.click()（原生相機）
  │
  └─ resolvedCaptureMode === 'pseudo-ar'（預設 auto 在支援 getUserMedia 裝置）
        └─ 呼叫 startPseudoCamera()
              ├─ 已在啟動中？→ return（防重複）
              ├─ 已在運行中？→ stopPseudoCamera() 後重啟
              ├─ 呼叫 navigator.mediaDevices.getUserMedia({ video: { facingMode: { ideal: 'environment' }, width: { ideal: 1920 }, height: { ideal: 1080 } }, audio: false })
              ├─ 成功：設 isPseudoCameraLive = true，videoEl.srcObject = stream，play()
              │     └─ 【新增】開啟全螢幕 overlay（見 3.3）
              └─ 失敗：pseudoCameraError = '無法啟動相機...'，stopPseudoCamera()，fallback 到 file 模式
```

### 3.3 全螢幕 / 橫式鎖定

**目標**：Pseudo AR 相機啟動後，以一個 `position: fixed; inset: 0` 的 overlay 蓋過整個畫面，提供更大的取景框，並作為 AR 相機預覽的容器。

**實作方式（待實作）**：
- 新增 Vue ref：`isCameraOverlayOpen = ref(false)`
- `startPseudoCamera()` 成功後設 `isCameraOverlayOpen = true`
- Overlay 內容：
  - `<video>` 全螢幕（object-fit: cover），顯示 live 鏡頭串流
  - **AR 疊加層組**（詳見 §3.4）：即時渲染於 `<video>` 之上
  - 右上角「✕ 關閉」按鈕（關閉後 stopPseudoCamera，清除 AR 疊加層）
  - 右上角「AR 開關」按鈕（切換 `isArPreviewEnabled`，隱藏 / 顯示所有 AR 疊加層）
  - 底部工具列：「拍照」按鈕（觸發 capturePhotoFromPseudoCamera）、縮放控制（見 §3.5）
- 嘗試橫式鎖定：
  ```javascript
  try {
    await screen.orientation.lock('landscape');
  } catch (e) {
    // iOS / 不支援：靜默忽略，不影響主流程（見 §4.1）
  }
  ```
- 關閉 overlay 時解除鎖定：`screen.orientation.unlock()`（若有支援）

### 3.4 AR 相機預覽疊加層規格

#### 3.4.1 功能定位

**核心需求**：「打開相機希望可以將明信片 AR 呈現在相機畫面上」——即在 live 鏡頭畫面上即時顯示明信片 AR 元素，讓使用者在拍照前就能預覽最終明信片構圖效果。

#### 3.4.2 AR 疊加層一覽

全螢幕相機 overlay 啟動後，在 `<video>` 之上以絕對定位依序渲染以下 AR 層：

| 層級（z-index 由低至高） | 名稱 | 描述 | Live 相機顯示 | Pending 截圖顯示 |
|---|---|---|---|---|
| z-[2] | **明信片導引框（Postcard Guide Frame）** | 以半透明白色描邊標示出明信片照片區的裁切比例（`aspect-ratio: 1.45/1`），輔助使用者對齊構圖 | ✅ 顯示 | ✅ 顯示（確認截圖框架） |
| z-[4] | **漸層遮罩（Gradient Overlay）** | 上深下淡黑色漸層，模擬最終明信片效果，讓文字疊加更易閱讀 | ✅ 顯示（透明度 60%） | ✅ 顯示 |
| z-[6] | **標題 / 地點文字預覽（Text Preview）** | 以目前使用者輸入的標題（`title`）與地點（`location`）值，以明信片樣式顯示於左上角（含 text-shadow）；若欄位為空則不顯示 | ✅ 顯示 | ✅ 顯示 |
| z-[8] | **皮克敏疊加層（Pikmin Overlay）** | 以目前使用者選取的皮克敏（`selectedPikminItems`）依照 `getPikminOverlayStyle()` 定位顯示 | ✅ 顯示 | ✅ 顯示 |

#### 3.4.3 可見性規則

```
isArPreviewEnabled = true（預設開啟）
  └─ AR 疊加層組整體顯示（v-show="isArPreviewEnabled"）

isArPreviewEnabled = false（使用者透過 AR 開關隱藏）
  └─ 所有 AR 疊加層隱藏（v-show="false"）
  └─ 僅顯示原始鏡頭畫面 + 導引框（導引框視為構圖輔助，可獨立保留，不受開關控制）

isPseudoCameraLive = false（相機未啟動 / 已停止）
  └─ 所有 AR 疊加層不渲染（v-if="isPseudoCameraLive"）
```

#### 3.4.4 拍照行為與截圖策略

**截圖策略：拍下純鏡頭原始畫面（Raw Frame），不含 AR 元素。**

- `capturePhotoFromPseudoCamera()` 僅對 `<video>` 元素執行 `drawImage`，canvas 上不疊加任何 AR 元素。
- AR 元素（漸層、文字、皮克敏）繼續由明信片 UI 的絕對定位疊加層渲染，與目前已實作行為一致。
- **理由**：AR 元素已透過 UI 層疊加，拍入照片底圖會造成重複疊加；此外保留原始底圖可讓使用者後續調整皮克敏位置 / 文字內容，靈活性更高。
- **例外**（可作為未來選項）：若使用者明確要求「匯出含 AR 的合成照片」，可在匯出 PNG 時另行執行 canvas 合成，但此為獨立功能，本次不實作。

#### 3.4.5 AR 預覽開關行為

- 全螢幕 overlay 右上角顯示「👁 AR」按鈕（icon + 文字）。
- 點擊後切換 `isArPreviewEnabled`（`true` ↔ `false`）。
- 按鈕狀態反饋：AR 開啟時按鈕高亮（如 `ring-2 ring-white`）；關閉時按鈕半透明。
- AR 開關狀態不持久化（每次開啟相機預設為開啟）。

### 3.5 縮放（Pinch-to-Zoom）

**目前狀態**：未實作，`<video>` 無縮放控制。

**計畫（待實作）**：
優先嘗試 Camera API zoom constraint：
```javascript
const track = stream.getVideoTracks()[0];
const capabilities = track.getCapabilities();
if (capabilities.zoom) {
  await track.applyConstraints({ advanced: [{ zoom: zoomValue }] });
}
```
若不支援，退回以 CSS `transform: scale(zoomValue)` 搭配 `overflow: hidden` 實現視覺縮放（不影響截圖解析度）。

手勢：
- `touchstart` / `touchmove` 偵測雙指距離變化計算 zoomValue
- 提供 `+` / `-` 按鈕作為備用

### 3.6 重拍流程

**目前行為**：`capturePhotoFromPseudoCamera()` 截圖後呼叫 `stopPseudoCamera()`，相機停止。若要重拍需再按「開始相機」。

**計畫（待實作）**：
```
capturePhotoFromPseudoCamera()
  └─ 設 photo = dataURL（截圖預覽）
  └─ 【不立即 stopPseudoCamera】
  └─ 顯示「確認 / 重拍」modal 或 inline 預覽區
        ├─ 使用者點「確認使用」→ stopPseudoCamera()，關閉 overlay，photo 保留
        └─ 使用者點「重拍」→ 清除 photo，相機繼續 live，使用者可再次拍照
```

---

## 4. 風險與降級策略

### 4.1 iOS 不支援 `screen.orientation.lock`

- **問題**：iOS Safari 不實作 `screen.orientation.lock()`，呼叫後丟 DOMException。
- **降級**：以 `try/catch` 包裹，失敗時靜默忽略。
- **替代方案**：告知使用者「請手動橫持裝置以獲得最佳體驗」（可透過偵測 `screen.orientation.type` 在直式時顯示提示）。
- **驗收**：iOS 上不應出現 uncaught error，相機仍可正常使用。

### 4.2 瀏覽器不支援 Fullscreen API

- **問題**：部分舊版行動瀏覽器不支援 `requestFullscreen`。
- **降級**：全螢幕 overlay 以 `position: fixed; inset: 0; z-index: 9999` 實作，**不依賴** Fullscreen API，故不受影響。
- **注意**：iOS Safari 的 PWA 模式可使用 fullscreen；一般瀏覽器模式限制較多，故選擇 CSS overlay 方案。

### 4.3 `getUserMedia` 失敗

| 原因 | 現有處理 | 補充建議 |
|------|----------|----------|
| 使用者拒絕相機權限 | `pseudoCameraError = '無法啟動相機，請確認相機權限...'` ✅ | 加入引導文字說明如何重設權限 |
| 裝置無相機 | 同上錯誤訊息 | 可透過 `enumerateDevices()` 預先偵測 |
| 瀏覽器不支援 getUserMedia | `supportsUserMedia()` 回傳 false → 自動 fallback 到 file 模式 ✅ | 無需額外處理 |
| HTTPS 限制（非安全環境） | getUserMedia 在 http 下被 block | 部署時確保 HTTPS；本機 localhost 例外 |
| 串流斷線（track ended） | 未處理 | 【建議】監聽 `track.addEventListener('ended', stopPseudoCamera)` |

### 4.4 截圖解析度問題

- `capturePhotoFromPseudoCamera()` 使用 `videoEl.videoWidth / videoHeight`，在低階裝置可能低於要求。
- 目前 getUserMedia 請求 `ideal: 1920x1080`，實際解析度由裝置決定。
- 若截圖後解析度不足，應於截圖後顯示提示，不阻擋使用者繼續。

### 4.5 AR 疊加層渲染效能

- **問題**：皮克敏圖片 + 文字 + 漸層等多層 DOM 元素在 live 相機畫面上持續渲染，可能在低階手機上造成掉幀或過熱。
- **降級策略**：
  - AR 疊加層使用 CSS `will-change: transform` + `transform: translateZ(0)` 強制 GPU 合成，減少重繪開銷。
  - 若偵測到裝置效能不足（可透過 `requestAnimationFrame` 計算 FPS < 20），自動關閉皮克敏疊加層，僅保留導引框與漸層。
  - 使用者可透過 AR 開關手動關閉 AR 預覽以提升相機流暢度。
- **驗收**：中階 Android（如 Snapdragon 695）上相機 live 畫面應維持 ≥ 24fps。

### 4.6 AR 疊加層尺寸比例相容性

- **問題**：全螢幕 overlay 的寬高比與明信片照片區（aspect-ratio: 1.45/1）不同，導引框需正確計算在全螢幕畫面中的位置與大小。
- **解法**：導引框以 CSS `aspect-ratio: 1.45/1` + `max-height: 85vh` + `margin: auto` 居中顯示，模擬明信片照片區的真實比例；不依賴固定 px 值。
- **注意**：`<video>` 使用 `object-fit: cover`，導引框應與 video 渲染區域對齊而非整個 viewport。

---

## 5. 驗收標準

### 5.1 相簿入口
- [ ] 點擊「開啟相簿」可開啟系統圖庫，選圖後明信片照片區正確顯示。
- [ ] `resolvedCaptureMode === 'file'` 時，「開啟相機」觸發原生相機（file input capture）。

### 5.2 Pseudo AR 相機開啟
- [ ] `resolvedCaptureMode === 'pseudo-ar'` 時，點擊「開始相機」後 `<video>` 顯示後鏡頭 live 串流。
- [ ] `isPseudoCameraStarting` 期間按鈕顯示「啟動中...」且不可重複點擊。
- [ ] 若 getUserMedia 失敗，顯示 `pseudoCameraError` 文字，模式自動切換為 file。

### 5.3 全螢幕 overlay（待實作後驗收）
- [ ] 相機啟動成功後，全螢幕 overlay 覆蓋整個視窗（`position: fixed; inset: 0`）。
- [ ] Overlay 內「✕」按鈕可關閉並停止相機。
- [ ] Android Chrome 上成功鎖定橫式方向；iOS Safari 上無 uncaught error，仍可使用相機。

### 5.4 拍照與重拍（待實作後驗收）
- [ ] 點擊「拍照」後顯示截圖預覽與「確認使用 / 重拍」選項。
- [ ] 點「重拍」後截圖清除，相機繼續 live（不需重新點「開始相機」）。
- [ ] 點「確認使用」後相機停止，照片正確設入 `photo`，明信片照片區顯示截圖。

### 5.5 縮放（待實作後驗收）
- [ ] 支援 Camera API zoom 的裝置可透過雙指縮放改變鏡頭焦距。
- [ ] 不支援的裝置可透過 CSS transform scale 縮放視覺畫面。
- [ ] `+` / `-` 按鈕可調整縮放層級（備用方案）。

### 5.6 降級行為
- [ ] `supportsUserMedia()` 回傳 false 的裝置，`resolvedCaptureMode` 自動為 `'file'`（現有邏輯，需迴歸確認）。
- [ ] `screen.orientation.lock` 拋例外時，相機功能不受影響。

### 5.7 AR 相機預覽（待實作後驗收）

**AR 疊加層顯示**：
- [ ] 開啟全螢幕相機後，明信片導引框（aspect-ratio: 1.45/1 半透明白框）自動顯示於 live 畫面上。
- [ ] 漸層遮罩（上深下淡）覆蓋於 live 畫面，模擬明信片最終效果。
- [ ] 若使用者已輸入標題 / 地點，文字以明信片樣式即時顯示於 live 畫面左上角。
- [ ] 若使用者已選取皮克敏，皮克敏依 `getPikminOverlayStyle()` 即時顯示於 live 畫面。

**AR 預覽開關**：
- [ ] 全螢幕 overlay 右上角有「AR 開關」按鈕，點擊可切換 AR 疊加層的顯示 / 隱藏。
- [ ] AR 關閉時，僅顯示導引框與純鏡頭畫面；AR 開啟時，所有 AR 元素恢復顯示。
- [ ] 按鈕狀態與 `isArPreviewEnabled` 一致（高亮 = 開啟，半透明 = 關閉）。

**截圖行為**：
- [ ] 點擊「拍照」後，`photo` 資料為純鏡頭原始畫面（不含 AR 元素的 canvas dataURL）。
- [ ] 截圖後，明信片 UI 中 AR 元素（漸層、文字、皮克敏）依舊正確疊加顯示於 `photo` 之上（沿用現有 UI 層疊加邏輯）。
- [ ] AR 開關狀態不影響截圖結果（截圖永遠為原始畫面）。

**效能與相容性**：
- [ ] 中階 Android 裝置上 live AR 預覽不低於 24fps（目視判斷無明顯卡頓）。
- [ ] 低效能裝置上 AR 元素可透過開關手動關閉，不影響拍照核心功能。

---

## 6. 未決議題（需產品確認）

### ✅ 6.1 「AI 相機模式」定義（已解決）
**結論**：「AI 相機模式」一詞已棄用，以「AR 相機預覽（AR Camera Preview）」取代。  
需求明確為：「打開相機希望可以將明信片 AR 呈現在相機畫面上」，即在 live 鏡頭畫面疊加明信片導引框、漸層、文字、皮克敏等 AR 元素供構圖預覽，無額外 AI 辨識功能。  
詳細規格見 §3.4。

### ✅ 6.2 「開啟相機」按鈕的最終行為（已解決）
**結論**：「開啟相機」按鈕在 `resolvedCaptureMode === 'pseudo-ar'` 時，直接觸發全螢幕 Pseudo AR 相機（含 AR 預覽）。  
「開啟相簿」與「開啟相機」維持分開入口，不整合。

### ✅ 6.3 疊加層在相機模式的顯示時機（已解決）
**結論**：皮克敏疊加層、漸層遮罩、標題 / 地點文字均在 live 相機畫面上即時顯示（`isArPreviewEnabled = true` 時），讓使用者構圖時即可預覽最終明信片效果。截圖時不將 AR 元素烘入照片底圖，由 UI 層另行疊加（見 §3.4.4）。

### 6.4 縮放值的持久化
**待確認**：使用者在相機模式調整的縮放值，拍照後是否應套用於明信片的照片顯示（background-size 調整）？

### 6.5 橫式鎖定的 UX 文字
**待確認**：iOS 無法自動鎖定橫式時，提示文字的中文措辭與顯示條件（是否僅在直式時才提示）。