# AR 拍照導入計畫：相機拍照時可直接使用 AR 方式拍攝目標物

## 背景與目標
- 目前專案已可透過上傳照片產生明信片，並用 `html2canvas` 匯出 PNG。
- 需求是讓使用者在「拍照當下」可直接用 AR 方式取景目標物，而不是先拍完再匯入。
- 本計畫採務實路線：先建立跨平台可用的拍照基線，再做 Android WebXR 增強，並保留既有匯入流程做最終備援。

## 現況（專案目前拍照/匯出流程）
1. 拍照/選圖入口：`<input type="file" accept="image/*" capture="environment">`。
2. `handlePhotoUpload` 使用 `FileReader.readAsDataURL`，把照片寫入 `photo` 狀態。
3. 預覽區以 `<img v-if="photo" :src="photo">` 顯示照片，並疊加標題、地點、皮克敏圖層。
4. 匯出流程 `handleDownload`：
   - `waitForPostcardRenderStability()` 等待字型/圖片穩定
   - `html2canvas(postcardRef, { scale: 2, useCORS: true })`
   - `canvas.toDataURL('image/png')` 下載
5. 現有可視為「file input 優先」流程；尚未有即時相機串流與 AR 取景模式。

## 平台可行性結論（Android vs iOS）
### Android（Chrome）
- 在相容裝置可使用 WebXR（`immersive-ar`）作為 AR 增強路徑。
- 仍需保留降級：若裝置/權限不符，回退到 pseudo-AR 或 file upload。

### iOS（Safari）
- Safari 對 WebXR AR 不可當作主路徑，需使用 fallback。
- 建議以 `getUserMedia + canvas` 提供 pseudo-AR（相機畫面 + 導引/疊圖 + 拍照凍結）。

### 統一策略
- 採 **progressive enhancement**：
  1. WebXR（可用時）
  2. pseudo-AR（`getUserMedia + canvas`）
  3. file upload（既有流程）

## 技術選項比較表
| 選項 | 支援平台 | 實作成本 | 體驗 | 與現有流程整合 |
| --- | --- | --- | --- | --- |
| WebXR AR（immersive-ar） | Android Chrome（相容裝置） | 中高 | 真 AR 取景最佳 | 需新增 AR 模組，但可把拍照結果輸回 `photo` |
| pseudo-AR（getUserMedia + canvas） | Android/iOS 主流瀏覽器 | 中 | 非真 AR，但可即時取景與疊圖 | 與現有 `photo` + `html2canvas` 最容易銜接 |
| file upload（現況） | 幾乎全部瀏覽器 | 低（已存在） | 最穩定但非即時 AR | 已完整實作，作為最終備援 |

## 分階段實作計畫
### Phase 1：跨平台基線（pseudo-AR 先落地）
目標：先讓 iOS/Android 都能在網頁內直接開相機拍照並回填現有流程。

- 新增 `cameraMode` 狀態：`webxr | pseudo-ar | file`。
- 新增能力偵測：
  - `supportsWebXRAR()`
  - `supportsUserMedia()`
- 新增 pseudo-AR UI：
  - `<video>` 顯示即時相機串流
  - 疊加取景框/簡易對齊層（CSS overlay）
  - 拍照按鈕把當前影像畫到 `<canvas>`，輸出 `dataURL` 寫回 `photo`
- 保留既有 `<input type="file">` 按鈕作手動降級。
- 匯出維持現況：仍走 `html2canvas`，避免一次改動過大。

### Phase 2：Android WebXR 增強
目標：在支援裝置提供更自然的 AR 取景。

- 僅在 `supportsWebXRAR()` 成功時顯示「AR 模式」入口。
- 建立 WebXR session 啟停與錯誤處理（權限拒絕、session 中斷）。
- 提供最小可用 AR 互動（例如目標對齊提示/簡易 reticle）。
- 拍照結果仍需回填到 `photo`（若 WebXR 擷取不穩定，立即回退 pseudo-AR 擷取）。
- 確保使用者可隨時切回 pseudo-AR 或 file upload。

### Phase 3：穩定化與上線準備
目標：降低回歸風險，完成跨平台驗收。

- 補齊裝置矩陣驗證：Android Chrome（支援/不支援 WebXR）、iOS Safari。
- 強化錯誤訊息與導引文案（權限、裝置不支援、相機占用）。
- 效能調校：控制相機解析度、拍照 canvas 尺寸，避免記憶體尖峰。
- 將模式選擇預設為「自動」，並可手動覆寫。

## 風險與備援
1. **WebXR 裝置相容差異**：同為 Android 也可能不可用。  
   備援：自動降級到 pseudo-AR。
2. **iOS 權限/播放限制**（需要 HTTPS、使用者手勢啟動）。  
   備援：相機啟動失敗時回到 file upload。
3. **相機串流與匯出畫面不一致**。  
   備援：拍照後固定回填 `photo`，後續預覽/匯出統一走既有 postcard DOM。
4. **效能與發熱問題**（長時間開啟相機）。  
   備援：離開拍照模式即 `stop()` tracks，限制解析度與重繪頻率。

## 驗收標準
- Android Chrome（支援 WebXR 裝置）：可進入 AR 模式拍照，完成後可正常下載明信片。
- iOS Safari：可使用 pseudo-AR 拍照並完成明信片下載。
- WebXR/相機權限失敗時：可明確提示並可改走 file upload，流程不中斷。
- 既有流程不回歸：
  - file input 仍可上傳
  - `html2canvas` 匯出成功
  - 檔名規則維持 `pikmin-postcard-{timestamp}.png`

## 下一步
1. 先實作 Phase 1 的能力偵測與 pseudo-AR 拍照（最低可用版本）。
2. 完成後進行 Android/iOS 實機驗證並修正權限與相容問題。
3. 再導入 Phase 2 的 WebXR 增強，並保持可隨時降級。
