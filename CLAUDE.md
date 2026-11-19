# Voxel-flight 專案摘要（體素飛行調校台）

> 這份文件給接手的 Claude Code 對話與日後的自己閱讀。開始工作前先讀完，再看「給 Claude Code 的工作規則」。

- 網址：https://wade-etins.github.io/voxel-flight/
- 儲存庫：https://github.com/wade-etins/voxel-flight （分支 `main`，GitHub Pages 從根目錄發佈）
- 擁有者：Wayne Sun（偏好繁體中文說明、直接的答案、清楚的修正步驟）
- 狀態（2026-11-18）：可玩的展示版，已安裝在 iPhone / iPad 主畫面

---

## 1. 這是什麼

一個像素風的飛行小遊戲，靈感來自一支 Instagram Reel（「No 3D engine, no polygons, 32 KB of code」）。
整個遊戲是一個 `index.html`，只用 Canvas 2D，**沒有 3D 引擎、沒有外部函式庫、沒有圖片或音檔素材**，所有畫面與聲音都由程式即時產生。

主要功能：

| 功能 | 說明 |
|---|---|
| 試玩 | 三種視角：俯視、追尾、座艙（含 HUD 與儀表） |
| 四個參數分頁 | 戰機、地形、操控、座艙；每個參數有滑桿、建議範圍與調整順序建議 |
| 倒帶檢視 | 保留最近 1200 幀（約 20 秒），可倒帶、逐幀、慢動作（0.05×–2×） |
| 重算 | 用同一組錄下的輸入、以新的操控參數重新模擬，比較手感 |
| 參數檔 | 存成情境（瀏覽器本機）、匯出／匯入 JSON、可只套用部分組別 |
| 五個場景 | 群島、城市、火山區、沙漠、北極極地（預設：北極極地） |
| 全螢幕 | 解析度依螢幕比例調整（橫向看得更廣），上方顯示即時數值 |
| 背景音樂 | 原創合成器循環樂句，WebAudio 即時合成，預設開啟 |
| 主畫面 App | PWA：manifest、圖示、service worker 離線快取 |

---

## 2. 檔案

| 檔案 | 用途 |
|---|---|
| `index.html` | 整個遊戲（HTML + CSS + JS，約 79 KB） |
| `manifest.webmanifest` | 主畫面 App 設定（名稱「體素飛行」、全螢幕、圖示） |
| `sw.js` | Service worker：有網路抓最新版並更新快取，沒網路用快取 |
| `icon-180.png` | iPhone 主畫面圖示（apple-touch-icon） |
| `icon-192.png`、`icon-512.png` | 一般 PWA 圖示 |
| `README.md` | GitHub 預設說明（尚未改寫） |

---

## 3. 程式架構（`index.html` 內的 `<script>`）

程式碼依編號分段，註解為繁體中文。用 LabVIEW 比喻：整個遊戲是一個 While Loop，每圈「讀輸入 → 更新狀態 → 重畫」。

| 段落 | 內容 | 重點函式／變數 |
|---|---|---|
| 1. 參數定義 | 四組參數的範圍、預設、建議文字、預設組合 | `SCHEMA`、`TUNED`、`P`、`applyParams()` |
| 2. 共用工具 | 解析度、亂數、雜湊 | `W/H`（預設 320×220）、`B`（偽 3D 緩衝區）、`N=1024` 高度圖、`CELL=2`、`VOXW=0.5` |
| 3. 程序地形 | 分形噪聲 → 高度圖 → 依場景規則上色；雲圖 | `fbm()`、`THEMES`、`buildPal()`、`genWorld()`、`genClouds()` |
| 4. 體素戰機 | 用方塊定義戰機，依姿態即時旋轉投影 | `buildJet()`、`basis()`、`camBasis()`、`drawJet()` |
| 5. 畫面 | 俯視、追尾、座艙三種渲染 | `renderOverhead()`、`renderVoxel()`（Voxel Space 算法）、`renderChase()`、`renderCockpit()` |
| 6. 飛行模型 | 輸入 → 傾角 → 轉彎；自動拉起、高度上限 | `step()`、`newState()` |
| 7. 錄影與輸入 | 每幀快照、倒帶、重算；滑鼠／觸控／鍵盤 | `REC`、`MAXREC=1200`、`resim()`、`readInput()` |
| 8. 主迴圈 | requestAnimationFrame | `frame()`、`updateUI()` |
| 9. 介面 | 分頁、參數面板、參數檔、全螢幕 | `showTab()`、`buildPanel()`、`applyPreset()`、`setFs()`、`fitRes()` |
| 10. 音樂與 PWA | 合成器樂句、service worker 註冊 | `BGM`、`firstGesture()` |

其他約定：

- 網址 hash 切換分頁：`#play`、`#jet`、`#terrain`、`#flight`、`#cockpit`、`#params`
- 瀏覽器儲存（localStorage）鍵：`vfl-params`（目前參數）、`vfl-scenes`（情境）、`vfl-bgm`（音樂開關）
- 參數改動後需重建的資料以 `regen` 標記：`world`（地形）、`clouds`（雲）、`jet`（戰機）
- 新增一個參數：只要在 `SCHEMA` 加一筆，面板、匯出、匯入會自動跟上

---

## 4. 目前的預設參數（Wayne 試飛後確定，2026-11-16）

寫在 `index.html` 的 `TUNED` 物件。修改預設值時改這裡，不要改 `SCHEMA` 裡的 `def`。

```json
{
 "jet": {"len":24,"span":22,"sweep":0.6,"fin":5,"size":1,"shades":4,"outline":true,"flame":1.7,"trail":65,"body":"#3a87fe","accent":"#f2b632","canopy":"#6fe0ea"},
 "terrain": {"theme":"arctic","seed":7,"sea":0.52,"scale":5,"rough":0.62,"terrace":0,"hScale":0.4,"fov":70,"far":1100,"detail":0.01,"fog":0.1,"camDist":70,"camH":20,"camLag":2.9,"camRoll":0.35,"zoom":1.6,"cloud":0.15,"cloudAlt":150,"wind":8},
 "flight": {"cruise":90,"boost":180,"accel":1.5,"maxBank":60,"rollRate":4,"turnRate":1.2,"maxPitch":30,"pitchRate":3,"barrel":0.85,"dead":0.08,"minClear":12,"ceiling":190},
 "cockpit": {"fov":92,"hud":true,"hudColor":"#5dff8a","ladder":10,"panel":0.25,"canopy":true,"shake":0.9,"radar":360}
}
```

另外：自動飛行預設開啟（碰畫面即轉手動），音樂預設開啟（瀏覽器規定第一次點擊後才出聲）。

---

## 5. 已做的決定與原因

| 決定 | 原因 |
|---|---|
| 不用 Three.js，全部 Canvas 2D | 符合原作「無 3D 引擎」的精神，單一檔案、載入快 |
| 戰機用程式定義的體素模型 | 任意滾轉角度連續平滑，不需手繪多張 sprite |
| 低解析度（320×220）再放大 | 像素風，且 iPhone 上效能足夠（一幀約 3–4 ms） |
| 地面優先於高度上限 | 高度上限 190 低於最高山頂時，避免穿山 |
| 俯視時雲畫成半透明網點 | 避免整片雲蓋住地面和戰機 |
| 背景音樂用原創合成樂句 | 不使用《捍衛戰士》主題曲 MIDI，避免版權問題 |
| 圖示用遊戲內的藍色戰機配夕陽 | 不仿電影標誌，維持原創 |
| 部署用 GitHub Pages + PWA | 免費、不需 Mac、可裝到主畫面並離線遊玩 |

---

## 6. 已知限制與待辦

已知限制：

- iPhone Safari 不一定允許網頁進入系統全螢幕；從主畫面開啟時才完整全螢幕
- iPhone 側邊靜音開關打開時，網頁音樂不會出聲
- 城市場景沒有碰撞，低飛會被推到樓頂上方，無法在大樓間穿梭
- 情境只存在單一裝置的瀏覽器；跨裝置要用匯出的 JSON
- 參數匯出只能複製，不能下載成檔案

可考慮的下一步：

- [ ] 改寫 `README.md` 為中文專案說明
- [ ] 實機確認 iPhone / iPad 的音樂、全螢幕、橫向版面
- [ ] 城市場景的大樓碰撞與穿梭
- [ ] 遊戲性：檢查點圈圈、計時、分數（若日後要上架 App Store，這類內容有助通過審核）
- [ ] 音效：引擎聲、翻滾音效

---

## 7. 自動推送更新的工作流程（可套用到其他專案）

### 原理

```
在 Claude Code 修改檔案 → git commit → git push 到 main
        → GitHub Pages 自動重新部署（約 1–2 分鐘）
        → 網址更新；主畫面 App 下次有網路開啟時抓到新版
```

GitHub Pages 本身就會在每次推送到 `main` 後自動部署，不需要另外設定 GitHub Actions。

### 第一次設定（每個新專案做一次）

1. 在 GitHub 建立 Public 儲存庫，上傳初始檔案（至少一個 `index.html`）
2. Settings → Pages → Source 選「Deploy from a branch」，Branch 選 `main`、`/ (root)`，Save
3. 在 Claude Code 開啟新對話時選這個儲存庫（或請 Claude 連接 `擁有者/儲存庫名`）
4. 在儲存庫根目錄放一份 `CLAUDE.md`（可直接用本文件改名），讓每次新對話自動讀到專案脈絡

### 每次更新

1. 對 Claude Code 說明要改什麼
2. Claude 修改、測試，並把變更內容告訴你
3. 你確認後說「推送」，Claude 執行 commit 與 push
4. 1–2 分鐘後重新整理網頁確認

### 建議的對話開場句

> 請先讀 Voxel-flight.md（或 CLAUDE.md）。我想要＿＿＿。改完後先測試並告訴我改了什麼，等我確認再推送到 main。

---

## 8. 給 Claude Code 的工作規則

1. **語言**：用繁體中文回覆；程式註解也用繁體中文
2. **推送前必須取得 Wayne 確認**：先說明改了哪些檔案、改了什麼，得到同意才 commit / push
3. **維持單一檔案、無外部依賴**：不引入函式庫、圖片或音檔，除非 Wayne 同意
4. **改了 `index.html` 或任何快取檔案時，把 `sw.js` 的 `CACHE` 版本號加一**（目前 `voxel-flight-v2`），確保主畫面 App 換到新版
5. **修改預設值改 `TUNED`**，不要改 `SCHEMA` 的 `def`
6. **推送前的檢查**：
   - 語法：把 `<script>` 內容抽出後執行 `node --check`
   - 畫面：用無頭 Chrome（Playwright）開啟並截圖，確認五個場景與三種視角沒有錯誤
7. **不使用受版權保護的素材**（電影配樂、標誌、角色）
8. **commit 訊息**用簡短中文或英文說明改了什麼，例如 `新增城市場景的大樓碰撞`

---

## 9. 開發紀錄

| 日期 | 內容 |
|---|---|
| 2026-11-16 | 評估 Reel 可行性；建立第一版：三種視角、四個參數分頁、倒帶逐幀、參數檔 |
| 2026-11-16 | 套用 Wayne 的試飛參數；新增全螢幕、iPhone/iPad 橫向支援、全螢幕數值列；新增城市、火山區、沙漠、北極極地場景 |
| 2026-11-16 | 預設場景改為北極極地 |
| 2026-11-16 | 加入 PWA（manifest、圖示、離線快取）與原創背景音樂；部署到 GitHub Pages |
| 2026-11-16 | 自動飛行與音樂預設開啟 |
| 2026-11-18 | 儲存庫連上 Claude；建立本摘要文件 |
