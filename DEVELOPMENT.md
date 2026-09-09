# 製作文件（DEVELOPMENT）

單車 GPS 降水地圖的製作與維護文件。騎士看的操作說明在 [`README.md`](README.md)；
原始開發計劃在 [`plans/2026-09-05-單車GPS降水地圖.md`](plans/2026-09-05-單車GPS降水地圖.md)（歷史文件）。

---

## 0. 現況（2026-09 暫停點）

| 項目 | 狀態 |
| --- | --- |
| 線上網址 | <https://ymeroom.github.io/biking-gps-weather/>（GitHub Pages，已部署，正常運作） |
| 原始碼 | 單一 `index.html`，無 build step，約 540 行 |
| git remote | `origin` → `github.com/ymeroom/biking-gps-weather.git` |
| 開發狀態 | v1 功能完成，**暫停開發**。商業化評估脈絡見 §9。 |

---

## 1. 這是什麼

騎車時放在手機架上的網頁：藍點跟著 GPS 移動，地圖疊上**日本氣象庁（JMA）官方雷達回波圖**，
時間軸可往前拉到未來約 2 小時，並依行進方向與速度推算「你到時會在哪」畫一個橘色幽靈點，
直接看那個未來位置會不會下雨。像 Tesla 車機導航雨雲圖的單車版。**只涵蓋日本境內**（見 §8）。

---

## 2. 架構總覽

- **單檔** `index.html`：HTML + CSS + JS 全部內嵌，無 build step、無打包工具。
- **外部相依只有 Leaflet 1.9.4**（unpkg CDN，SRI 雜湊鎖定）。
- **底圖**：OpenStreetMap 標準圖磚。傍晚模式用純 CSS `filter` 反轉圖磚，不動雷達圖層。
- **疊圖**：JMA 雷達圖磚（見 §3）。
- **PWA**：`manifest.webmanifest` + `icon.svg` + `apple-mobile-web-app-*` meta，可「加到主畫面」像 App。
- **無後端、無資料庫、無 API key**。所有資料在瀏覽器端直接向 JMA 抓。
- **部署** = 把檔案推到 GitHub、Pages 從 `main` 根目錄發佈。

### 檔案清單

| 檔案 | 作用 |
| --- | --- |
| `index.html` | 全部的程式（HTML/CSS/JS 內嵌） |
| `manifest.webmanifest` | PWA 設定 |
| `icon.svg` | App icon |
| `README.md` | 使用說明（騎士看的） |
| `DEVELOPMENT.md` | 本檔（製作／維護文件） |
| `plans/2026-09-05-*.md` | 原始開發計劃（歷史） |

---

## 3. 資料源：JMA 雷達圖磚

### 端點

Base：`https://www.jma.go.jp/bosai/jmatile/data`

**時刻索引（JSON）**

| URL（相對 base） | 內容 | 網格 / 間隔 |
| --- | --- | --- |
| `nowc/targetTimes_N1.json` | 高解像度降水ナウキャスト — 觀測 | 250m，過去約 60 分、5 分一格 |
| `nowc/targetTimes_N2.json` | 高解像度降水ナウキャスト — 預報 | 250m，未來約 60 分、5 分一格 |
| `rasrf/targetTimes.json` | 降水短時間予報 | 1km，未來 1–15 小時、每小時一格 |

**圖磚**

- `nowc/{basetime}/none/{validtime}/surf/hrpns/{z}/{x}/{y}.png` — hrpns，圖磚到 z10
- `rasrf/{basetime}/none/{validtime}/surf/rasrf/{z}/{x}/{y}.png` — rasrf，圖磚只到 z8

程式裡 `maxNativeZoom` 依 `kind` 分別設 z10 / z8。

### CORS（2026-09-09 實測確認）

JMA 對帶 `Origin` 標頭的請求會回 **`Access-Control-Allow-Origin: *`**（JSON 與圖磚皆是，連 404 回應都帶）。所以：

- 可跨域 `fetch()` targetTimes JSON ✓
- 可把圖磚畫進 `<canvas>` 再 `getImageData()` 取樣（「你頭上目前」功能靠這個）✓
- `index.html` 對圖磚 `<img>` 設 `crossOrigin='anonymous'` 是**正確且必要**的

> ⚠️ `plans/2026-09-05-*.md` 第 33 行寫「不需要 CORS、JMA 沒回 Access-Control 標頭」——
> 那是實作前用 `curl -I`（未帶 `Origin`）得到的舊結論，**已過時**。以本節為準。

### 這是非公開端點

`bosai/jmatile` 是 JMA 官網「雨雲の動き」頁面自己在用的端點，**沒有公開文件、沒有服務保證、沒有 rate limit 說明**。
格式哪天變了要跟著修（見 §7）。性質等同逆向別人網站的內部 API。

---

## 4. 程式結構導覽（`index.html`）

依原始碼的分隔註解，由上到下：

| 區塊 | 起始行（約） | 內容 |
| --- | --- | --- |
| 常數 | 127 | `JMA_BASE`、refresh 間隔、色階 `JMA_SCALE` |
| 狀態 | 147 | 單一 `state` 物件（pos / frames / idx / playing / wakeLock…） |
| 地圖 | 161 | Leaflet 初始化、OSM 底圖、深色底圖 CSS filter |
| GPS 定位 | 197 | `watchPosition`、方向平滑、top-bar chip 更新 |
| 推算位置幽靈點 | 258 | 大圓航行外推 `destPoint()` / `updateGhost()` |
| JMA 雷達圖層 | 289 | `tileUrl()`、`loadRadarIndex()`、`showFrame()` |
| 你頭上目前 | 389 | canvas 取樣、`classifyRGB()` |
| 播放 / 控制 | 442 | 播放迴圈、Wake Lock、圖例 |
| 綁定 / 啟動 | 501 | 事件綁定、初始化呼叫 |

---

## 5. 關鍵設計決策

不寫下來，六個月後會被重新踩坑的東西。

1. **「現在」= 最後一張觀測影格，不是時間上最接近 `now` 的影格**（`nowFrameIdx()` ~L353）。
   N2 的第一格其實是 +2～5 分的預報；用「最接近 now」會跳到預報格。刻意取最後一張非預報。

2. **自動更新不搶走使用者正在看的未來時刻**（`loadRadarIndex()` ~L329–340，commit `a3c989d`）。
   每 2.5 分重抓 targetTimes 後：本來停在「現在」→ 跳到新的「現在」；正在看某未來格 →
   對齊到新資料裡時間最接近的那格。`wasAtNow` 判斷是這段核心。

3. **雙緩衝圖磚圖層防閃白**（`showFrame()` ~L364–372）。切換時間時新圖層疊上、舊圖層等新的
   `load` 事件才移除，另有 1500ms 保險 timeout。播放時連續切格才不會每格閃白底。

4. **方向用 sin/cos 的 EMA 平滑**（`smoothHeading()` ~L197）。直接平滑角度會在 359°→1° 跳變出錯；
   改成分別平滑 sin、cos 再 `atan2` 還原，避開 wraparound。α = 0.4。

5. **時間字串當 UTC 解析、顯示走 JST**（`parseJmaTime()` ~L191 用 `Date.UTC(...)`；顯示用
   `Intl.DateTimeFormat` timeZone `Asia/Tokyo`）。看起來像 bug，其實是對的：JMA 的
   basetime/validtime 字串是 UTC。

6. **播放中不做「你頭上目前」取樣**（`showFrame()` ~L385 `if(!state.playing)`）。canvas 取樣每格要
   下載 + decode 一張圖磚，播放時每 0.9 秒一次太耗電。停在某格才算。

7. **未來上限 3h、但對騎士只承諾 2h**（`FUTURE_LIMIT_MS` ~L131）。rasrf 每小時一格，多抓 1–2 格才能
   保證畫面填滿到 +2h。時間軸實際可能拉到 +3h 附近，屬正常，不是 bug。

---

## 6. 「你頭上目前」取樣機制（風險最高的一段）

流程（`updateOverhead()` ~L406）：

1. GPS 座標 → 固定 zoom（hrpns z10 / rasrf z8）的圖磚 `x/y` + 磚內小數位置。
2. `new Image()` 載入那張圖磚（`crossOrigin='anonymous'`）。
3. 畫進 canvas，`getImageData()` 取那 1 個像素的 RGBA。
4. `classifyRGB()` ~L396：跟 `JMA_SCALE` 的 8 個色階做**最近 RGB 距離**比對，距離平方 < 2500
   才算命中，否則視為無降水／未知。

> ⚠️ **`JMA_SCALE`（~L135）是實測 hrpns 圖磚逆向出來的 RGB，不是官方色表。**
> JMA 若調整色階：不會報錯，會**靜默誤判**（中雨講成小雨、有雨講成無雨），
> 目前沒有任何機制偵測或警告。這是整個程式最脆弱的地方。

`maxNativeZoom` 的 rasrf z8 / hrpns z10 分界出現在**兩處**——`showFrame()` ~L368 與
`updateOverhead()` ~L412——改一個要同步改另一個。

---

## 7. 失敗模式表

端點無公開文件，以下是「會壞的東西 → 你怎麼發現 → 去哪修」：

| 會壞的東西 | 徵兆 | 修哪裡 |
| --- | --- | --- |
| targetTimes JSON 結構改變 | top-bar「JMA：失敗」，toast 顯示錯誤訊息 | `loadRadarIndex()` 的 `mk()` 與過濾邏輯 |
| 圖磚 URL 路徑格式改變 | 地圖上沒有雷達疊圖，但 chip 顯示正常 | `tileUrl()` ~L289 |
| JMA 色階調整 | 無錯誤，「你頭上目前」開始講錯（見 §6） | `JMA_SCALE` ~L135，需重新取樣校色 |
| rasrf「予報 run」挑選失準 | 未來 60 分–2h 那段沒圖或跳掉 | `loadRadarIndex()` ~L316–317：挑「最新且含未來 validtime 的 basetime」，邏輯偏脆 |
| GitHub Pages referrer / mixed-content 擋圖磚 | 部署後圖磚 403 / 不載入 | tile layer 加 `referrerpolicy="no-referrer"` |
| Leaflet CDN 掛掉 | 整頁白 | 改本機打包 Leaflet 或換 CDN（SRI 雜湊要一起換） |

---

## 8. 資料涵蓋範圍的硬限制

**只涵蓋日本境內。** JMA 雷達合成圖由日本各地雷達拼成，最西南的雷達在**石垣島**（24.4°N, 124.2°E）。

關於「用 JMA 看台灣天氣」（曾評估過）：

- 台灣**北海岸**（金山、萬里，離石垣島約 270km）勉強在雷達邊緣，對流強時看得到回波。
- 但 270km 距離波束已在 5–6km 高空，**低層的鋒面小雨、毛毛雨整片漏掉**。
- 環騎台北**南段**（新店、木柵、二格山，約 24.9°N，離石垣島 >320km）**完全在範圍外**。
- 風櫃嘴、五指山這些最容易被大屯山系地形雨打到的路段，剛好是 JMA 覆蓋最弱處。

**結論：JMA 資料不足以支撐台北的產品。** 台灣要用中央氣象署（CWA）開放資料：

| CWA 產品 | 用途 |
| --- | --- |
| `O-A0058-003` 雷達整合回波圖-臺灣 | 全台地面級、約 10 分更新、單張 georeferenced PNG（疊法比 JMA 圖磚簡單） |
| 定量降水預報 / 臨近預報 | 0–2h，CWA 官方版 nowcast |
| `F-D0047` 鄉鎮天氣預報 | 臺北市／新北市各區逐 3 小時降雨機率（行前規劃用） |

需在 CWA 開放資料平臺免費申請 API 授權碼。

---

## 9. 商業化 / 授權 / 法規備忘

暫停點的決策脈絡，之後要撿起來時看這節。

### 授權（JMA）

- **JMA ≠ JWA。** JWA（日本気象協会 / tenki.jp）是另一個賣資料的法人，跟本專案無關。
- JMA 資料**無 API key、無費用**。
- JMA 內容利用規約 = 政府標準利用規約（等同 CC-BY 4.0），唯一義務是**標示出處**。
- 已滿足：`index.html` ~L104 圖例註明「資料：気象庁 高解像度降水ナウキャスト／降水短時間予報」。

### 法規（日本）

- 《気象業務法》§17：在日本以事業形式「發布預報」需**予報業務許可**。
- 單純原樣轉貼 JMA 官方預報圖磚 → 一般不算。
- 但「幽靈點 + 你到時會在這、那裡在下雨」的**客製化預測講法**，一旦收費就進灰色地帶。
  收費前應向 JMA / 行政書士確認。

### 法規（台灣，若改用 CWA）

- 《氣象法》§17、§18：經營氣象預報業務需中央氣象署許可；非依法不得發布氣象預報。
- 引用氣象署發布的預報 → 允許。
- 自製降水外推 + 收費 → 同樣灰色地帶。

### 商業化評估結論（2026-09）

- 護城河薄：薄殼包免費政府資料，容易被複製。
- 差異化在「**沿路線 × 時間的降水推算**」，不在「顯示雷達」。
- 日本多日旅行版：市場太窄（一年用兩次）。
- 環騎台北版：使用場景集中、每週用、痛點真實 → 商業合理性較高，但**必須換 CWA 資料源**。
- 定價方向：一次性買斷 NT$60–120，走「贊助 / 請喝咖啡」框架，不做訂閱。
- iOS 上架唯一跑不掉的固定成本 = Apple 開發者帳號 US$99/年。
- **未動工。** 要推進的下一步：(1) CWA 資料實測 (2) 問氣象署許可問題 (3) PTT bike 版 / 環北社團問 10–20 人願不願付費。

---

## 10. 本機開發

```bash
cd "D:\biking gps weather"
python -m http.server 8000
# 瀏覽器開 http://localhost:8000
```

- 定位（geolocation）只在 **HTTPS** 或 `http://localhost` 下有效，雙擊 `file://` 會被瀏覽器擋。
- 電腦上測不到真實 GPS 移動；要測藍點跟隨 / 幽靈點，用手機開 Pages 網址，或用
  Chrome DevTools 的 **Sensors** 面板灌假座標。
- Wake Lock 需 HTTPS + 使用者互動後才給（localhost 也算安全語境）。

---

## 11. 部署

- GitHub repo：`ymeroom/biking-gps-weather`（`origin` 已設）。
- GitHub Pages：Settings → Pages → Source = `main` 分支 `/ (root)`。
- 每次 `git push` 到 `main`，1–2 分鐘後線上自動更新。
- 線上網址：<https://ymeroom.github.io/biking-gps-weather/>
- `.gitignore` 已排除 `.remember/` 等雜物；push 前確認沒帶進測試截圖。

---

## 12. 未做 / 未來（YAGNI 清單）

- 日本以外的雷達（要換 CWA / RainViewer）。
- 沿實際路線（會轉彎）推算未來位置 —— 目前只等速直線外推。
- 離線快取雷達圖磚。
- 航跡記錄、GPX 匯出、降雨語音／震動警報。
- 商業化功能（路線匯入、補給點、導航 deep link）。
