# A-PixPic — Handover to Claude Code

## 一句話
手機優先的 4 階灰階 Bayer 抖色濾鏡 webapp。三種來源:相機 / 圖 / GLB 模型,全部進同一個濾鏡。純靜態,部署在 Cloudflare Pages(GitHub 連動),無 build step。

## 狀態(2026-10-19,換環境交接)
- `main` 在 `56c1b54`(PR #1 已 squash merge)。`sw.js` cache 名現為 `pixpic-v2`。
- 本輪已修(headless Chromium + fake camera + reduced-motion 驗過 9 項,無 pageerror):
  1. `[ SHOOT ]` 僅在有有效畫面時啟用(`live` 旗標,`frame()` 內 `setLive(ok)`),相機被擋 / PHOTO 未選圖不再存出黑圖。
  2. `startCam()` 用 `camSeq` 請求序號,`stopCam()` 會讓進行中的請求過期;快速 FLIP / 切來源不洩漏 stream。
  3. `prefers-reduced-motion` 下 `#flash` 直接 `display:none`。
  4. 結果頁(`#sheet`):開啟時焦點進 `CLOSE`、背景 `header/main/#ctl` 設 `inert`、Esc 關閉、關閉後焦點回 `SHOOT`。
- **未做**:iPhone 實機測試(相機權限、Web Share 存圖、加入主畫面)。
- **未做 · 部署被擋**:Cloudflare Pages 連線與 `apixpic.adamliu.uk` 綁定。上一個環境沒有 Cloudflare 工具、沒有 `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID`,且網路政策擋 `apixpic.adamliu.uk` / `*.pages.dev`(proxy 回 502,不代表網站掛)。線上目前是否存在、是哪個版本:**未驗證**。
- Adam 已授權「所有部署項目」。新環境若有 token 與放行網路,直接執行下方「部署」。

## 部署(Cloudflare Pages,未執行)
- Dashboard 路線:Workers & Pages → Pages → Connect to Git → `LIUYUTSO/A-PixPic`;Production branch `main`;Build command 留空;Output directory `/`;Custom domains 加 `apixpic.adamliu.uk`。
- CLI 路線(需 Secrets `CLOUDFLARE_API_TOKEN`〔Account → Cloudflare Pages: Edit〕、`CLOUDFLARE_ACCOUNT_ID`,網路放行 `api.cloudflare.com`):`npx wrangler pages project create apixpic --production-branch main` → `npx wrangler pages deploy . --project-name apixpic --branch main`。
- 驗證:`curl https://apixpic.adamliu.uk/sw.js` 應含 `pixpic-v2`。token 只放環境 Secrets,不貼對話。

## 檔案
- `index.html` — 全部邏輯與樣式(單檔,ES module)。測試 hook:`window.__pixpic`。
- `icon-512.png` / `icon-192.png` / `apple-touch-icon.png` — 用同一套濾鏡畫的老相機(Bayer 4×4、4 階、整數倍放大)。產生腳本不在 repo;要改 icon 需重寫(見下方濾鏡規格)。
- `sw.js` — network-first 的最小 offline shell,cache 名 `pixpic-v1`。改 `index.html` 後若要強迫更新,bump 這個版本號。

## 濾鏡規格(JS 與 icon 必須一致)
luma(0.299/0.587/0.114) → `(v-.5)*contrast+.5` → Bayer 4×4 閾值 `(B+.5)/16` → `q=clamp(floor(v*3+t),0,3)` → 調色盤 `#000 #555 #AAA #FFF`。
格寬 `RES=[96,160,240]`,格高 = 寬×4/3。顯示用整數倍 nearest-neighbor 放大;輸出 PNG 倍率 `max(4, ceil(1080/W))`。

## 設計決策(不要翻案,除非 Adam 說)
- 套用 adam-ui **Mode E · Reductionist Mono**:全大寫、等寬字(Martian Mono)、零圓角、零陰影、零 accent、純黑白、無動畫(僅 opacity)。控制項是純文字 `[ ]`,不是按鈕外觀。
- 介面**英文**(Adam 明確要求);對話與文件用繁體中文(台灣用語)。
- 3D 用 Three.js **0.160.0**,jsdelivr importmap,**首次進 3D 才動態 import**(CDN 掛掉不影響相機/圖)。支援 GLB/glTF、Draco(gstatic decoder)、Meshopt。
- 刻意沒有框架、沒有 build。不要引入 Next.js/Tailwind/GSAP——這是「簡單 HTML」專案。

## 已知限制 / 待辦(依優先序)
1. **iPhone 實機驗證**:相機 `getUserMedia`、`navigator.share({files})`、standalone 模式下的版面(safe-area)。
2. 3D 需網路(CDN)。若要離線:把 three + addons vendoring 進 repo 並改 importmap,同步更新 `sw.js` 快取清單。
3. 檢視區是 3:4 置中,大螢幕/高螢幕上下留白偏多。**Adam 已明說先不動**,除非他再提。
4. 手機直式鎖定只寫在 manifest,Safari 分頁內不會鎖。
5. 無 iOS 相簿直存:iOS 靠分享面板「儲存影像」;桌面走 `<a download>`。
6. 效能:PHOTO / CAM 畫面沒變也每幀重跑濾鏡(耗電),可只在來源或參數變動時重算。

## 環境注意
- 不要用 `pkill -f <pattern>`:pattern 會比對到自己的 shell 指令列,把 shell 殺掉(exit 144),後面的 commit / push 不會執行。停背景 server 請記 PID 再 `kill`。
- 合併過的 PR 不要重用:從最新 `main` 重開分支再做新改動。

## 測試方式(headless,可重現)
Playwright + Chromium 旗標 `--use-fake-device-for-media-stream --use-fake-ui-for-media-stream --use-gl=swiftshader --enable-unsafe-swiftshader`;把 `cdn.jsdelivr.net/npm/three@0.160.0/**` route 到本機 `node_modules/three`,字型請求 abort(預期會有一則 `ERR_FAILED` console error,正常)。流程:CAM→SHOOT→PHOTO 載圖→3D demo→以 `GLTFExporter` 產 GLB 餵 `#file`→餵壞檔確認錯誤訊息。
本機起靜態伺服器即可:`python3 -m http.server`(相機需 `localhost` 或 HTTPS)。

## Adam 的工作規則(摘要)
- 先結論後理由;不吞錯誤、不假裝成功;不替 Adam 命名;不問已固定的事(棧/語言/風格)。
- 終端機執行要有驗證與重試、輸出清楚。
- 要 Adam 實測的東西,收斂成**單一可執行視窗**,不要他組裝多個檔。
- 視覺優先思考(看留白與墨跡邊界,不從數值關係起手)。
- 無障礙硬指標(對比、reduced-motion、focus 可見)可否決審美;其餘 Adam 說了算,警告最多一次。
- 無 AI 紫藍漸層。
