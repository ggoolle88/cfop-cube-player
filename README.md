# 魔術方塊 CFOP 學習 App：規劃與原型

**線上試用：** https://ggoolle88.github.io/cfop-cube-player/ （手機、平板、電腦皆可）

## 目前檔案

| 路徑 | 內容 |
|---|---|
| [prototype/cube-player.html](prototype/cube-player.html) | 自製 Three.js 引擎：3x3x3 模型、記號解析器、動畫引擎、播放／暫停／拖曳／速度控制，並內建 PLL 21、OLL 57、F2L 41 案例圖選擇器；可切換 2 階（Ortega 解法：OLL 7、PBL 8）。公式與圖示皆以模擬器驗證，直接用瀏覽器開啟即可 |
| [prototype/formula-detail.html](prototype/formula-detail.html) | 以 cubing.js `<twisty-player>` 製作的「公式詳情頁」原型 |
| [data/schema/cfop-case.schema.json](data/schema/cfop-case.schema.json) | CFOP 案例 JSON Schema |
| [data/algorithms/samples.json](data/algorithms/samples.json) | 範例資料：T-Perm、Sune (OLL 27)、F2L 1 |

## 技術選型

**建議：Expo (React Native + TypeScript)，3D 播放器放在 WebView 內執行。**

- cubing.js 的 `twisty-player` 是 Web Component，無法在 RN 原生層直接執行；放進 WebView 後可以直接使用它的 3D/2D 播放、stickering（OLL/PLL 遮色）與 alg viewer，同一份程式碼也能做成網頁版。
- 若需要 cubing.js 做不到的自訂效果（手勢轉層、特殊高亮），可用 `prototype/cube-player.html` 的自製 Three.js 引擎，同樣跑在 WebView 中。之後若要改成原生渲染，可移植到 `@react-three/fiber/native` + `expo-gl`，引擎本身只依賴 three.js。
- 列表、測驗、進度等 UI 頁面用 RN 原生元件撰寫；展開圖用 `react-native-svg` 依 JSON `diagram` 繪製，不必開 WebView，列表捲動才會順。

其他選項的取捨：

- **Flutter**：沒有成熟的魔術方塊函式庫，最後仍要嵌 WebView 或自己寫 3D。
- **Unity**：3D 很強，但這個 App 大部分是列表、表單、測驗等 UI，Unity 做這類介面成本高，App 也會很大。
- **roofpig**：多年未維護，依賴舊版 three.js 與 jQuery，不建議作為新專案基礎。
- **授權**：cubing.js 採 MPL-2.0 / GPL-3.0 雙授權，上架前請確認各套件授權是否符合商業用途。

周邊套件：expo-router（導覽）、Zustand（狀態管理）、expo-sqlite（學習進度與測驗紀錄）、react-native-webview、react-native-svg。

## 專案資料夾規劃

```
cube-app/
├─ app/                         # expo-router 頁面
│  ├─ (tabs)/index.tsx          # 今日練習（間隔重複佇列）
│  ├─ (tabs)/library.tsx        # 公式庫：F2L / OLL / PLL 分頁＋篩選
│  ├─ (tabs)/progress.tsx       # 進度統計
│  ├─ case/[id].tsx             # 公式詳情頁
│  ├─ quiz/[mode].tsx           # 測驗：看圖選名稱 / 看名稱打公式 / 計時
│  └─ notation.tsx              # 符號說明（每個記號附小動畫）
├─ src/
│  ├─ components/               # CaseCard、TopDiagram (SVG)、PlayerControls…
│  ├─ cube-view/                # CubeWebView.tsx：RN ↔ WebView 訊息橋接
│  ├─ features/progress/        # SRS 排程 (SM-2)、學習狀態
│  ├─ features/quiz/
│  ├─ data/                     # JSON 載入、由 schema 產生的 TS 型別
│  └─ store/
├─ web-player/                  # WebView 內的網頁，用 Vite 打包成單一 HTML 放進 assets
│  └─ src/ parser.ts · engine.ts · twisty.ts · bridge.ts
├─ data/
│  ├─ schema/cfop-case.schema.json
│  ├─ algorithms/f2l.json · oll.json · pll.json
│  └─ notation.json
└─ scripts/validate-data.ts     # CI：schema 驗證、步數重算、steps 串接需等於 alg
```

WebView 橋接訊息（`postMessage`）：

- RN → Web：`{type:"load", alg, setupAnchor:"end", stickering}`、`{type:"play"}`、`{type:"pause"}`、`{type:"seek", move}`、`{type:"speed", value}`
- Web → RN：`{type:"progress", move, total, playing}`，RN 端據此同步高亮「步驟說明」

## 公式詳情頁 UI/UX

```
┌───────────────────────────┐
│ ‹  T-Perm  T 型交換   PLL·14步 │  頂部導覽：名稱、分類、步數、機率
├───────────────────────────┤
│ [頂面圖]             3D│2D │  左上縮圖＝JSON diagram；右上切換視角
│                           │
│        3D 方塊（可旋轉）     │  約佔螢幕 40%；初始狀態＝「案例」
│                           │
│  R U R' U' R' F R2 …      │  公式字串，目前步驟高亮，點擊可跳到該步
│  ━━━━━━━●━━━━━━━━━━        │  進度條（可拖曳）
│    ⏮   ◀   [ ▶ ]   ▶   ⏭    │
│   0.5× │ 1× │ 2× │ 4×       │
├───────────────────────────┤
│ 步驟說明│辨識特徵│指法│替代公式 │  分頁
│ ① R U R' U'  [Sexy Move]  │  點片段 → 動畫只播放該段
│ ② R' F                    │
├───────────────────────────┤
│ 加入每日練習          ( ●) │  固定在底部
│ 學習狀態  未學│學習中│已掌握  │
│ [      開始辨識測驗       ] │
└───────────────────────────┘
```

設計重點：

1. 預設從「案例狀態」開始（cubing.js 的 `experimental-setup-anchor="end"`），播放結束時頂層剛好還原，符合練習時的實際情境。
2. 預設黃色朝上（setup 加上 `z2`），與多數 CFOP 教材一致；OLL/PLL 使用 stickering 把無關的貼紙變灰。
3. 步驟片段、指法筆記與 3D 時間軸連動：播放到哪一步就高亮哪一段，點擊任一段也會跳到對應時間點。
4. 練習區固定在底部，學習中隨時可以把案例加入練習清單。
