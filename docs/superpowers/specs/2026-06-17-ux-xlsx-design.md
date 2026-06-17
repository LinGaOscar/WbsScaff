# WBS Builder — UX 優化 + XLSX 匯出設計

**日期**：2026-06-17  
**範圍**：全部展開/收合、狀態統計、搜尋+狀態篩選、XLSX 匯出  
**不在範圍內**：localStorage、鍵盤快捷鍵、多專案管理

---

## 功能一：全部展開 / 收合按鈕

### 位置
Header `#header-actions` 內，「模板」按鈕後、`.sep` 後，「新增群組」前。

### 行為
- **⊞ 全展開**：`collapsed = {}; render();`
- **⊟ 全收合**：遍歷 `allFlat(tree)` 中所有 `hasChildren` 節點，設 `collapsed[id] = true`；`render()`

```js
function expandAll()  { collapsed = {}; render(); }
function collapseAll() {
  allFlat(tree).forEach(n => { if (n.children.length) collapsed[n.id] = true; });
  render();
}
```

### 樣式
使用現有 `.btn.btn-muted`，無需新增 CSS。

---

## 功能二：狀態統計（Stats 列新增一格）

### 顯示格式
Stats 列在「最大層級」後新增第 4 格：

```
完成 X/Y（Z%）
```

- Y = `allFlat(tree).length`（所有節點總數）
- X = `allFlat(tree).filter(n => n.status === 'done').length`
- Z = `Math.round(X / Y * 100)` %；Y 為 0 時顯示 `0%`

### 實作
`render()` 的 Stats 區段新增：
```js
const done = allNodes.filter(n => n.status === 'done').length;
const pct  = allNodes.length ? Math.round(done / allNodes.length * 100) : 0;
document.getElementById('s-done').textContent = `${done}/${allNodes.length}（${pct}%）`;
```

HTML 新增對應 `<div>`：
```html
<div><span class="stat-val" id="s-done">0/0（0%）</span> 完成</div>
```

---

## 功能三：搜尋 + 狀態篩選

### 全域狀態
```js
let filterText   = '';   // 關鍵字，toLowerCase 後比對
let filterStatus = 'all'; // 'all' | 'not-started' | 'in-progress' | 'done'
```

### UI 位置
Header `#header-actions` 內，最左側（現有按鈕群最前面），與其他按鈕用 `.sep` 分隔。

元件：
1. `<input type="text" id="filter-input" placeholder="🔍 搜尋節點…">` — 輸入時 `oninput` 更新 `filterText` 並 render
2. `<select id="filter-status">` — 選項：全部 / 未開始 / 進行中 / 完成；`onchange` 更新 `filterStatus` 並 render

### 過濾邏輯
在 `render()` 的 `vis.forEach` 迴圈中，對每個節點計算 `isMatch`：

```js
const textMatch   = !filterText || node.title.toLowerCase().includes(filterText);
const statusMatch = filterStatus === 'all' || (node.status || 'not-started') === filterStatus;
const isMatch     = textMatch && statusMatch;
```

不符合的節點 **暗化**（保留 DOM，降低 opacity），不隱藏，維持樹狀脈絡：
```js
row.style.opacity = isMatch ? '1' : '0.25';
row.style.pointerEvents = isMatch ? '' : 'none';
```

### CSS
```css
#filter-input {
  padding: 6px 10px; border-radius: 7px;
  border: 1px solid var(--border2); background: var(--surface);
  color: var(--text); font-size: 13px; font-family: inherit;
  width: 160px; outline: none; transition: border-color 0.15s;
}
#filter-input:focus { border-color: var(--accent); }
#filter-status {
  padding: 6px 8px; border-radius: 7px;
  border: 1px solid var(--border2); background: var(--surface);
  color: var(--text); font-size: 13px; font-family: inherit;
  cursor: pointer; outline: none;
}
```

### 手機版
`@media (max-width: 640px)` 搜尋框寬度改為 `100%`，select 也 `width: 100%`。

---

## 功能四：XLSX 匯出

### SheetJS 取得方式
從 jsDelivr CDN 下載 `xlsx.full.min.js`（約 1MB），以 `<script>` inline 嵌入 `index.html`，維持離線可用性。

下載指令：
```bash
curl -o /tmp/xlsx.full.min.js "https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"
```

嵌入位置：HTML `</body>` 前的 `<script>` 標籤**前**，獨立一個 `<script>` 標籤。

### 匯出函式 `exportXLSX()`
```js
function exportXLSX() {
  const nums = numbering(tree);
  const rows = [['編號','層級','工作項目','負責人','開始日期','結束日期','狀態']];
  const walk = (nodes, level=0) => nodes.forEach(n => {
    rows.push([nums[n.id], level+1, n.title, n.owner||'', n.startDate||'', n.endDate||'', STATUS_LABEL[n.status||'not-started']]);
    walk(n.children, level+1);
  });
  walk(tree);
  const ws = XLSX.utils.aoa_to_sheet(rows);
  // 欄寬自動調整
  ws['!cols'] = rows[0].map((_, i) => ({
    wch: Math.max(...rows.map(r => String(r[i]||'').length)) + 2
  }));
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'WBS');
  XLSX.writeFile(wb, 'wbs.xlsx');
  showToast('XLSX 已匯出');
}
```

注意：`exportXLSX` 直接使用全域 `STATUS_LABEL` 常數，與 `exportCSV` 保持一致。

### UI
Header 現有 `⬇ CSV/Excel` 按鈕旁新增 `⬇ XLSX` 按鈕（`.btn.btn-muted`）。

---

## CLAUDE.md 更新

完成後需更新：
1. 全域狀態加入 `filterText`、`filterStatus`
2. 函式表新增 `expandAll`、`collapseAll`、`exportXLSX`
3. 第 3 階段標記 XLSX 匯出為已完成
