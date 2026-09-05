# 單車 GPS 降水地圖

騎車時打開的手機網頁：藍點跟著你的 GPS 移動，地圖疊上**日本氣象庁（JMA）官方雷達回波圖**，
時間軸可往前拉到**未來 2 小時**，並依你的行進方向與速度推算「你到時會在哪」畫一個幽靈點，
直接看那個未來位置會不會下雨。像 Tesla 車機導航那個雨雲圖的單車版。

只涵蓋**日本境內**（資料源是 JMA）。

## 使用方式

1. 手機瀏覽器開 `https://ymeroom.github.io/biking-gps-weather/`（部署後）
2. 允許定位權限
3. 加到主畫面，之後點 icon 就能開
4. 建議帶行動電源 —— GPS + 螢幕常亮很吃電

> ⚠️ 定位功能只在 HTTPS 網址下有效。直接雙擊本機 `index.html`（`file://`）在手機上定位會被瀏覽器擋。
> 本機測試請用 `python -m http.server` 開 `http://localhost:8000`。

## 畫面說明

- **藍點＋三角形**：你目前的位置與行進方向；外圈是定位精度。
- **橘色虛線點**：推算的未來位置（時間軸拉到未來時才出現）。靜止時無法推算，會疊在現在位置上。
- **時間軸**：左邊是過去 1 小時的觀測回波，中間是現在，右邊是未來預報。
  - 未來 0–60 分：高解像度降水ナウキャスト（250m 網格，5 分一格）
  - 未來 60 分–2 小時：降水短時間予報（1km 網格，每小時一格，畫面較粗）
- **你頭上目前**：讀取雷達圖在你座標的回波值，直接告訴你現在（或所選時刻）頭頂的雨勢。
- 每 2.5 分鐘自動抓 JMA 最新資料。

## 技術

- 單一 `index.html`，無 build step。Leaflet 1.9.4 + OpenStreetMap 底圖。
- JMA 圖磚端點（官方網站自用、無公開文件）：
  - `https://www.jma.go.jp/bosai/jmatile/data/nowc/{basetime}/none/{validtime}/surf/hrpns/{z}/{x}/{y}.png`
  - `https://www.jma.go.jp/bosai/jmatile/data/rasrf/{basetime}/none/{validtime}/surf/rasrf/{z}/{x}/{y}.png`
  - 時刻索引：`nowc/targetTimes_N1.json`（觀測）、`nowc/targetTimes_N2.json`（60 分預報）、`rasrf/targetTimes.json`（短時間予報）
  - 圖磚與 JSON 皆回 `Access-Control-Allow-Origin: *`，可跨域 fetch 與 canvas 取樣。
- 若 JMA 改端點格式，`tileUrl()` 與 `loadRadarIndex()` 是唯一要改的地方。

## 不包含（未來可加）

- 日本以外的雷達資料
- 沿實際路線（會轉彎）推算未來位置 —— 目前只做等速直線外推
- 離線快取雷達圖磚
- 航跡記錄、GPX 匯出、降雨語音／震動警報

## 相關

行前規劃用的行程網站雷達頁（RainViewer、全球、精度較低）：
`riding.braintaiwan.com` 的 `radar-map.html`。這個工具是騎乘中即時使用、換 JMA 官方資料、獨立部署。
