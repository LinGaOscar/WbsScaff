# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案目標

單一離線 HTML 檔（`index.html`），無任何外部依賴，適用於氣隙（air-gapped）生產環境。
使用者可直接在瀏覽器開啟使用，不需安裝或伺服器。

## 開發方式

無建置步驟。直接以瀏覽器開啟 `index.html` 預覽即可。

```bash
open index.html        # macOS
start index.html       # Windows
```

修改後重新整理頁面（F5）即可看到效果。

## 核心限制

- **禁止引入任何 CDN 或 npm 套件**。若第三方套件必要（如 SheetJS），需手動 inline 進 HTML。
- CSS 色彩定義集中在 `:root` CSS Variables，修改主題只動 `:root`。
- ID 命名：`kebab-case`；JS 函式：`camelCase`。
- 最大樹狀層級為 5 層（level 0–4），新增子節點按鈕在 `level < 4` 才顯示。

## 架構：資料模型與渲染流程

**全域狀態**（`<script>` 頂部宣告）：

```js
let tree = [...];           // 樹狀資料
let collapsed = {};         // { [nodeId]: boolean }，記錄折疊狀態
let draggingId = null;      // 拖曳中的節點 id
let overId = null;          // 目前拖曳懸停的節點 id（drag-over 視覺提示用）
let expandedNotes = {};     // { [nodeId]: boolean }，備註列展開狀態
let dropPosition = null;    // 'before' | 'child'，dragover 時更新
```

**節點資料結構**：

```js
{ id: string, title: string, children: Node[], owner?: string, startDate?: string, endDate?: string, status?: 'not-started'|'in-progress'|'done', notes?: string }
```

`owner`/`startDate`/`endDate` 為選填欄位（節點列右側的「時間起訖／負責人」）。**行為差異**：
- 透過 `addRoot/addSibling/addChildNode` 建立的節點**不含**這三個欄位
- 透過 `injectIds()`（模板套用或 JSON 匯入）建立的節點這三個欄位**一律為空字串 `""`**
- 兩種情形 `node[field]` truthy 檢查與 `updateNodeField` 都能正確處理

日期顯示格式為 `MM/DD`（內部儲存也是 `MM/DD`，非 ISO 格式）。

**渲染管線**（每次資料變動後呼叫 `render()`）：

`render()` 每次都做**完整 DOM 重建**（無 diff、無 virtual DOM），任何資料異動都須呼叫一次；勿嘗試局部更新 DOM。

1. `numbering(tree)` — 遞迴產生 `{id: "1.2.3"}` 的編號 map
2. `allFlat(tree)` — 遞迴展平**所有**節點（含 level），用於計算統計數字
3. `flatten(tree)` — 展平**可見**節點（跳過 `collapsed[id]` 的子樹），用於 DOM 渲染
4. DOM 依照展平結果逐一建立 `.node-row` 元素

**主要 JS 函式對照**：

| 函式 | 用途 |
|------|------|
| `uid()` | 產生隨機 id |
| `findNode(nodes, id)` | 遞迴搜尋節點 |
| `removeNode(nodes, id)` | 不可變刪除，回傳 `[newTree, removedNode]` |
| `insertAfter(nodes, targetId, newNode)` | 插入目標節點之後（同層） |
| `addChildTo(nodes, parentId, newNode)` | 插入指定節點的子列表末尾 |
| `updateNodeTitle(nodes, id, title)` | 不可變更新標題 |
| `updateNodeField(nodes, id, field, value)` | 不可變更新任意欄位（`owner`/`startDate`/`endDate`） |
| `startEdit(id, titleEl)` | 啟動標題行內編輯（`contenteditable`），Enter 或 blur 確認 |
| `startMetaEdit(id, field, el, placeholder)` | 啟動 meta 欄位編輯；日期欄位用 `<input type="date">` 並呼叫 `showPicker()` |
| `injectIds(nodes)` | 遞迴補 uid 並補齊 `owner/startDate/endDate`，用於套用模板或 JSON 匯入時重建節點 |
| `cycleStatus(id, current)` | 循環切換節點狀態（not-started→in-progress→done→not-started）並 render |
| `insertBefore(nodes, targetId, newNode)` | 插入目標節點之前（同層） |
| `isDescendant(nodes, ancestorId, targetId)` | 判斷 targetId 是否為 ancestorId 的後代（防循環拖曳） |
| `getNodeDepth(nodes, id, depth)` | 取得節點在樹中的深度（0-indexed） |
| `subtreeDepth(node)` | 取得節點子樹的最大高度（用於拖曳層級限制） |

**拖曳排序**：`dragstart/drop` 事件觸發 `removeNode` + `insertAfter`，目前僅支援同層重排；跨層拖曳為第 2 階段目標。

**WBS 模板**：`TEMPLATES` 物件（feature／system／project 三種）定義純資料樹（不含 id），`applyTemplate(key)` 以 `injectIds()` 補 id 後整棵取代 `tree`，確認框防止誤觸。新增模板請直接擴充 `TEMPLATES`，並在 `#tpl-panel` 加對應 `.tpl-card`。

> ⚠️ 注意：早期版本曾有「AI 生成」功能（直接呼叫 Anthropic API），已於 commit `32b4194` 移除，目前程式碼**沒有**任何 AI 相關邏輯。`建置計畫書.md` 中提及 AI 生成的段落已過時，請勿依此重新引入。

**匯出 CSV**：含 UTF-8 BOM（`﻿`），確保 Excel 直接開啟中文不亂碼；欄位含編號、層級、工作項目、負責人、開始/結束日期。

**配色慣例**：`:root` 中區分兩組色彩——`--bg/--surface/--card` 等深色變數用於 Header／模板面板（外層），`--canvas/--on-canvas*` 等淺色變數用於 `#tree` 卡片內部（淺色卡片設計）。新增 UI 元件時依其所在區塊（外層深色 vs. 樹狀卡片內淺色）挑選對應變數，勿混用。

**RWD**：`@media (max-width: 640px)` 統一管理手機版樣式（Header 換行、meta 欄位換行至節點第二行、動作按鈕常駐顯示）。新增節點列相關樣式時，桌面與手機版規則需同步檢查。

## 第 2 階段功能（已完成）

- 跨層拖曳（移動節點到不同父節點）✓
- 節點狀態標籤（未開始 / 進行中 / 完成，點擊循環切換）✓
- 節點備註欄位（展開列，blur 儲存）✓

## 第 3 階段（選用）

XLSX 匯出（SheetJS inline bundle）、列印 PDF 樣式、甘特圖 SVG 預覽。
