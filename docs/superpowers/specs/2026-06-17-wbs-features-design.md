# WBS Builder — 第 2 階段功能設計

**日期**：2026-06-17  
**範圍**：跨層拖曳、節點狀態標籤、節點備註欄位  
**不在範圍內**：localStorage 自動存檔、鍵盤快捷鍵、多專案管理

---

## 資料模型

節點物件新增兩個選填欄位，沿用現有 `owner/startDate/endDate` 慣例：

```js
{
  id: string,
  title: string,
  children: Node[],
  owner?: string,
  startDate?: string,   // MM/DD
  endDate?: string,     // MM/DD
  status?: 'not-started' | 'in-progress' | 'done',  // 新增
  notes?: string,                                     // 新增
}
```

**`injectIds()` 更新**：補上 `status: n.status || 'not-started'` 與 `notes: n.notes || ''`，確保模板套用與 JSON 匯入後欄位一律存在。

**CSV 匯出**：`exportCSV()` 增加「狀態」欄位（最後一欄），中文顯示（未開始／進行中／完成）。

---

## 功能一：節點狀態標籤

### UI 位置
`.node-title` 左側插入 `.status-badge` pill 標籤。

### 狀態對應

| 值 | 顯示文字 | 顏色變數 |
|----|---------|----------|
| `not-started` | 未開始 | `--muted2`（灰） |
| `in-progress` | 進行中 | `--yellow`（黃） |
| `done` | 完成 | `--green`（綠） |

### 互動
點擊標籤 → 循環切換（not-started → in-progress → done → not-started）→ 呼叫 `updateNodeField(tree, id, 'status', newStatus)` + `render()`。

### 樣式
```css
.status-badge {
  font-size: 11px; padding: 2px 7px; border-radius: 10px;
  cursor: pointer; flex-shrink: 0; font-weight: 600;
  border: 1px solid currentColor; transition: opacity 0.15s;
}
.status-badge:hover { opacity: 0.75; }
```

---

## 功能二：節點備註欄位（展開列）

### 結構
每個 `.node-row` 後方緊接一個 `.node-notes-row`，預設 `display: none`。

```
.node-row           ← 現有節點列
.node-notes-row     ← 新增，展開後顯示 <textarea>
```

### 展開/收合
`node-actions` 區新增備註按鈕 `💬`：
- 無備註：灰色圖示
- 有備註：帶顏色（`--accent`）圖示提示
- 點擊切換展開/收合，狀態存在 `expandedNotes = {}` 全域物件（`{ [nodeId]: boolean }`）

### 編輯
展開後顯示 `<textarea>`，行數依內容自動撐高（`rows="2"`，`resize: vertical`）。`blur` 時呼叫 `updateNodeField(tree, id, 'notes', value)` 儲存，不呼叫 `render()`（避免 textarea 閃爍失焦）。

### 縮排
`.node-notes-row` 的 `padding-left` 與對應節點列相同，視覺上對齊標題。

### 手機版
`@media (max-width: 640px)` 備註列全寬顯示，`textarea` 寬度 100%。

---

## 功能三：跨層拖曳

### 放置判斷規則（區域判斷）
拖曳懸停時，以 `getBoundingClientRect()` 計算游標相對目標列的位置：

- **上半部**（`clientY < rect.top + rect.height * 0.5`）→ 同層插入目標前（保留現有 `insertAfter` 邏輯，改為 `insertBefore`）
- **下半部**（`clientY >= rect.top + rect.height * 0.5`）→ 成為目標的最後子項（`addChildTo`）

### 視覺回饋
- 上半：目標列左側藍線（現有 `.drag-over` 樣式）
- 下半：整列底色加深 + 右側顯示縮排提示（`⤵` 文字）

新增 CSS class `.drag-over-child` 與 `.drag-over-before` 分別對應兩種狀態。

### 層級限制
執行 drop 前計算新深度：若超過 4 層，阻擋操作並呼叫 `showToast('已達最大層級（5 層）', true)`。

### 循環參照防護
drop 時若目標節點是被拖曳節點的後代，阻擋操作（否則會形成循環樹）。新增輔助函式 `isDescendant(nodes, ancestorId, targetId): boolean` 在 drop handler 中判斷。

### `insertBefore` 新增函式
現有只有 `insertAfter`，需新增 `insertBefore(nodes, targetId, newNode)` 對稱函式。

### 全域狀態新增
```js
let dropPosition = null;  // 'before' | 'child'，dragover 時更新
```

---

## CLAUDE.md 更新

完成實作後，需更新：
1. 節點資料結構加入 `status` 與 `notes` 欄位
2. 函式表新增 `insertBefore`
3. 全域狀態新增 `expandedNotes` 與 `dropPosition`
4. 第 2 階段功能表移除自動存檔、快捷鍵、多專案管理，標記已完成項目
