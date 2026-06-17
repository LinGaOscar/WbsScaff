# WBS Phase 2 Features Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在單一 `index.html` 新增三個功能：節點狀態標籤（點擊循環）、備註展開列、跨層拖曳。

**Architecture:** 所有修改集中於 `index.html` 的 `<style>` 與 `<script>` 區塊。資料層新增 `status` / `notes` 欄位至節點物件。渲染層在 `render()` 內新增對應 DOM 元素。跨層拖曳以 `getBoundingClientRect()` 判斷上/下半區域決定插入方式。

**Tech Stack:** 純 HTML / CSS / Vanilla JS，無任何外部依賴，無建置步驟。

---

## 檔案修改範圍

- **Modify:** `index.html` — 唯一檔案，以下任務皆修改此檔

---

## Task 1：全域狀態與資料模型

**Files:**
- Modify: `index.html:460-462`（全域狀態）
- Modify: `index.html:806-813`（`injectIds`）
- Modify: `index.html:774-781`（`applyTemplate`）
- Modify: `index.html:851-852`（`handleImport`）

- [ ] **Step 1：新增全域常數與狀態變數**

在 `index.html:462`（`let overId = null;` 後）插入：

```js
let expandedNotes = {};   // { [nodeId]: boolean }
let dropPosition = null;  // 'before' | 'child'

const STATUS_CYCLE = { 'not-started': 'in-progress', 'in-progress': 'done', 'done': 'not-started' };
const STATUS_LABEL = { 'not-started': '未開始', 'in-progress': '進行中', 'done': '完成' };
```

- [ ] **Step 2：更新 `injectIds()` 補齊新欄位**

將 `index.html:806-813` 的 `injectIds` 函式替換為：

```js
function injectIds(nodes, level=0) {
  return (nodes||[]).map(n => ({
    id: uid(), title: n.title || '未命名',
    owner: n.owner || '', startDate: n.startDate || '', endDate: n.endDate || '',
    status: n.status || 'not-started',
    notes: n.notes || '',
    children: injectIds(n.children, level+1)
  }));
}
```

- [ ] **Step 3：`applyTemplate()` 重置 `expandedNotes`**

將 `index.html:774-781` 的 `applyTemplate` 函式替換為：

```js
function applyTemplate(key) {
  if (!confirm('套用模板將取代現有 WBS，確定繼續？')) return;
  tree = injectIds(TEMPLATES[key]);
  collapsed = {};
  expandedNotes = {};
  document.getElementById('tpl-panel').classList.remove('open');
  render();
  showToast('模板已套用 ✓');
}
```

- [ ] **Step 4：`handleImport` 重置 `expandedNotes`**

在 `index.html` `handleImport` 函式內，找到：
```js
      tree = injectIds(data); // re-assign ids for safety
      collapsed = {};
```
替換為：
```js
      tree = injectIds(data);
      collapsed = {};
      expandedNotes = {};
```

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat: 新增 status/notes 資料欄位與全域狀態"
```

---

## Task 2：節點狀態標籤

**Files:**
- Modify: `index.html`（`<style>` 區塊末尾、`<script>` 函式區、`render()` 內）

- [ ] **Step 1：新增 CSS**

在 `index.html` `</style>` 前插入：

```css
  /* ── Status Badge ── */
  .status-badge {
    font-size: 11px; padding: 2px 8px; border-radius: 10px;
    cursor: pointer; flex-shrink: 0; font-weight: 600;
    border: 1px solid transparent; transition: opacity 0.15s;
    white-space: nowrap;
  }
  .status-badge:hover { opacity: 0.7; }
  .status-not-started { color: var(--muted2); border-color: var(--muted2); }
  .status-in-progress { color: var(--yellow); border-color: var(--yellow); }
  .status-done        { color: var(--green);  border-color: var(--green);  }
```

- [ ] **Step 2：新增 `cycleStatus()` 函式**

在 `// ── Edit ──` 區塊前插入：

```js
// ── Status ──
function cycleStatus(id, current) {
  tree = updateNodeField(tree, id, 'status', STATUS_CYCLE[current] || 'not-started');
  render();
}
```

- [ ] **Step 3：在 `render()` 的 Title 前插入狀態標籤**

找到 `render()` 內：
```js
      // Title
      const title = document.createElement('span');
```
在此行**前**插入：

```js
      // Status badge
      const status = node.status || 'not-started';
      const sBadge = document.createElement('span');
      sBadge.className = `status-badge status-${status}`;
      sBadge.textContent = STATUS_LABEL[status];
      sBadge.onclick = e => { e.stopPropagation(); cycleStatus(node.id, status); };
      row.appendChild(sBadge);
```

- [ ] **Step 4：開啟瀏覽器確認**

```bash
open index.html
```

驗證：每個節點列顯示灰色「未開始」標籤；點擊循環變「進行中」（黃）→「完成」（綠）→「未開始」（灰）。

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat: 新增節點狀態標籤（點擊循環切換）"
```

---

## Task 3：備註展開列

**Files:**
- Modify: `index.html`（`<style>` 區塊、`render()` 內）

- [ ] **Step 1：新增 CSS**

在 `</style>` 前插入：

```css
  /* ── Notes Row ── */
  .node-notes-row { display: none; }
  .node-notes-row.open { display: block; }
  .notes-textarea {
    width: 100%; background: #fff;
    border: 1px solid var(--canvas-border); border-radius: 6px;
    padding: 6px 10px; font-size: 13px; color: var(--on-canvas);
    font-family: inherit; resize: vertical; min-height: 52px;
    outline: none; transition: border-color 0.12s; display: block;
  }
  .notes-textarea:focus { border-color: var(--accent); }
  .act-btn.notes.has-notes { color: var(--accent); }

  @media (max-width: 640px) {
    .notes-textarea { font-size: 14px; }
  }
```

- [ ] **Step 2：在 `render()` 的 actions 區加入備註按鈕**

找到：
```js
      acts.appendChild(mkBtn('✕','del','刪除', () => deleteNode(node.id)));
```
在此行**前**插入：

```js
      const hasNotes = !!(node.notes && node.notes.trim());
      acts.appendChild(mkBtn('💬', 'notes' + (hasNotes ? ' has-notes' : ''), '備註', () => {
        expandedNotes[node.id] = !expandedNotes[node.id];
        const nr = document.querySelector(`.node-notes-row[data-id="${node.id}"]`);
        if (nr) nr.classList.toggle('open', !!expandedNotes[node.id]);
      }));
```

- [ ] **Step 3：在 `treeEl.appendChild(row)` 後插入備註列**

找到：
```js
      treeEl.appendChild(row);
```
在此行**後**插入：

```js
      // Notes row
      const notesRow = document.createElement('div');
      notesRow.className = 'node-notes-row' + (expandedNotes[node.id] ? ' open' : '');
      notesRow.dataset.id = node.id;
      notesRow.style.paddingLeft = (24 + node.level * 26 + 28) + 'px';
      notesRow.style.paddingRight = '12px';
      notesRow.style.paddingBottom = '6px';
      const ta = document.createElement('textarea');
      ta.className = 'notes-textarea';
      ta.placeholder = '新增備註…';
      ta.value = node.notes || '';
      ta.rows = 2;
      ta.addEventListener('blur', () => {
        tree = updateNodeField(tree, node.id, 'notes', ta.value.trim());
      });
      ta.addEventListener('click', e => e.stopPropagation());
      ta.addEventListener('dragstart', e => e.stopPropagation());
      notesRow.appendChild(ta);
      treeEl.appendChild(notesRow);
```

- [ ] **Step 4：開啟瀏覽器確認**

```bash
open index.html
```

驗證：節點 hover 時出現 💬 按鈕；點擊展開 textarea；輸入文字後點擊其他地方儲存；有備註的節點 💬 呈藍色；再次點擊收合。

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat: 新增節點備註展開列"
```

---

## Task 4：跨層拖曳

**Files:**
- Modify: `index.html`（`<style>` 區塊、`<script>` 函式區、`render()` 拖曳事件）

- [ ] **Step 1：新增 CSS 拖曳視覺回饋**

將 `index.html:104` 的現有樣式：
```css
  .node-row.drag-over { border-left-color: var(--accent) !important; background: #4e9af128; }
```
替換為：
```css
  .node-row.drag-over-before { border-left-color: var(--accent) !important; background: #4e9af128; }
  .node-row.drag-over-child  { border-left-color: var(--green)  !important; background: #3fb68e1a; }
```

- [ ] **Step 2：新增輔助函式**

在 `// ── Tree utils ──` 區塊的現有函式後（`updateNodeField` 之後）插入：

```js
function insertBefore(nodes, targetId, newNode) {
  return nodes.reduce((acc, n) => {
    if (n.id === targetId) acc.push(newNode);
    acc.push({ ...n, children: insertBefore(n.children, targetId, newNode) });
    return acc;
  }, []);
}

function isDescendant(nodes, ancestorId, targetId) {
  const anc = findNode(nodes, ancestorId);
  if (!anc) return false;
  return !!findNode(anc.children, targetId);
}

function getNodeDepth(nodes, id, depth=0) {
  for (const n of nodes) {
    if (n.id === id) return depth;
    const d = getNodeDepth(n.children, id, depth+1);
    if (d !== -1) return d;
  }
  return -1;
}

function subtreeDepth(node) {
  if (!node.children.length) return 0;
  return 1 + Math.max(...node.children.map(subtreeDepth));
}
```

- [ ] **Step 3：替換 `render()` 內的拖曳事件處理器**

找到並替換整個拖曳事件區塊（`// Drag events` 到 `treeEl.appendChild(row);` 之前）：

舊程式碼：
```js
      // Drag events
      row.addEventListener('dragstart', e => {
        draggingId = node.id;
        setTimeout(() => row.classList.add('dragging'), 0);
      });
      row.addEventListener('dragend', () => {
        draggingId = null; overId = null;
        document.querySelectorAll('.drag-over,.dragging').forEach(el => {
          el.classList.remove('drag-over','dragging');
        });
      });
      row.addEventListener('dragover', e => {
        e.preventDefault();
        if (draggingId === node.id) return;
        document.querySelectorAll('.drag-over').forEach(el => el.classList.remove('drag-over'));
        row.classList.add('drag-over');
        overId = node.id;
      });
      row.addEventListener('drop', e => {
        e.preventDefault();
        if (!draggingId || draggingId === node.id) return;
        const [without, moved] = removeNode(tree, draggingId);
        if (moved) tree = insertAfter(without, node.id, moved);
        draggingId = null; overId = null;
        render();
      });
```

替換為：
```js
      // Drag events
      row.addEventListener('dragstart', e => {
        draggingId = node.id;
        setTimeout(() => row.classList.add('dragging'), 0);
      });
      row.addEventListener('dragend', () => {
        draggingId = null; overId = null; dropPosition = null;
        document.querySelectorAll('.drag-over-before,.drag-over-child,.dragging').forEach(el => {
          el.classList.remove('drag-over-before','drag-over-child','dragging');
        });
      });
      row.addEventListener('dragover', e => {
        e.preventDefault();
        if (draggingId === node.id) return;
        document.querySelectorAll('.drag-over-before,.drag-over-child').forEach(el => {
          el.classList.remove('drag-over-before','drag-over-child');
        });
        const rect = row.getBoundingClientRect();
        dropPosition = e.clientY >= rect.top + rect.height * 0.5 ? 'child' : 'before';
        row.classList.add(dropPosition === 'child' ? 'drag-over-child' : 'drag-over-before');
        overId = node.id;
      });
      row.addEventListener('drop', e => {
        e.preventDefault();
        if (!draggingId || draggingId === node.id) return;
        if (isDescendant(tree, draggingId, node.id)) {
          showToast('無法拖曳到自身的子節點', true);
          draggingId = null; overId = null; dropPosition = null;
          return;
        }
        const [without, moved] = removeNode(tree, draggingId);
        if (!moved) { draggingId = null; overId = null; dropPosition = null; return; }
        const targetDepth = getNodeDepth(without, node.id);
        const newDepth = dropPosition === 'child' ? targetDepth + 1 : targetDepth;
        if (newDepth + subtreeDepth(moved) > 4) {
          showToast('已達最大層級（5 層）', true);
          draggingId = null; overId = null; dropPosition = null;
          return;
        }
        if (dropPosition === 'child') {
          tree = addChildTo(without, node.id, moved);
          collapsed[node.id] = false;
        } else {
          tree = insertBefore(without, node.id, moved);
        }
        draggingId = null; overId = null; dropPosition = null;
        render();
      });
```

- [ ] **Step 4：開啟瀏覽器確認**

```bash
open index.html
```

驗證：
- 拖曳節點到另一節點**上半部** → 左側藍線，放開後插入其前方同層
- 拖曳節點到另一節點**下半部** → 左側綠線，放開後成為其子項並展開
- 拖曳父節點到其後代 → toast 警告「無法拖曳到自身的子節點」
- 拖曳至超過第 5 層 → toast 警告「已達最大層級（5 層）」

- [ ] **Step 5：Commit**

```bash
git add index.html
git commit -m "feat: 跨層拖曳（上半插前、下半成子項）"
```

---

## Task 5：CSV 匯出更新 + CLAUDE.md 更新

**Files:**
- Modify: `index.html:823-838`（`exportCSV`）
- Modify: `CLAUDE.md`

- [ ] **Step 1：更新 `exportCSV()` 加入狀態欄位**

找到 `exportCSV` 函式，替換如下：

```js
function exportCSV() {
  const nums = numbering(tree);
  const rows = [['編號','層級','工作項目','負責人','開始日期','結束日期','狀態']];
  const walk = (nodes, level=0) => nodes.forEach(n => {
    rows.push([nums[n.id], level+1, n.title, n.owner||'', n.startDate||'', n.endDate||'', STATUS_LABEL[n.status||'not-started']]);
    walk(n.children, level+1);
  });
  walk(tree);
  const csv = rows.map(r => r.map(c => `"${String(c).replace(/"/g,'""')}"`).join(',')).join('\r\n');
  const bom = '﻿';
  const blob = new Blob([bom+csv], { type: 'text/csv;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href=url; a.download='wbs.csv'; a.click();
  URL.revokeObjectURL(url);
  showToast('CSV 已匯出（Excel 可直接開啟）');
}
```

- [ ] **Step 2：更新 CLAUDE.md 節點資料結構**

在 CLAUDE.md 的節點資料結構中，將：
```js
{ id: string, title: string, children: Node[], owner?: string, startDate?: string, endDate?: string }
```
替換為：
```js
{ id: string, title: string, children: Node[], owner?: string, startDate?: string, endDate?: string, status?: 'not-started'|'in-progress'|'done', notes?: string }
```

- [ ] **Step 3：更新 CLAUDE.md 全域狀態**

將全域狀態區塊：
```js
let tree = [...];
let collapsed = {};
let draggingId = null;
let overId = null;
```
替換為：
```js
let tree = [...];
let collapsed = {};
let draggingId = null;
let overId = null;
let expandedNotes = {};   // { [nodeId]: boolean }
let dropPosition = null;  // 'before' | 'child'
```

- [ ] **Step 4：更新 CLAUDE.md 函式對照表**

在函式表新增：

| 函式 | 用途 |
|------|------|
| `cycleStatus(id, current)` | 循環切換節點狀態並 render |
| `insertBefore(nodes, targetId, newNode)` | 插入目標節點之前（同層） |
| `isDescendant(nodes, ancestorId, targetId)` | 判斷 targetId 是否為 ancestorId 的後代（防循環拖曳） |
| `getNodeDepth(nodes, id, depth)` | 取得節點在樹中的深度（0-indexed） |
| `subtreeDepth(node)` | 取得節點子樹的最大高度（用於拖曳層級限制） |

- [ ] **Step 5：更新 CLAUDE.md 第 2 階段功能表**

將「第 2 階段待實作功能」區塊替換為：

```markdown
## 第 2 階段功能（已完成）

- 跨層拖曳（移動節點到不同父節點）✓
- 節點狀態標籤（未開始 / 進行中 / 完成）✓
- 節點備註欄位 ✓

## 第 3 階段（選用）

XLSX 匯出（SheetJS inline bundle）、列印 PDF 樣式、甘特圖 SVG 預覽。
```

- [ ] **Step 6：最終瀏覽器確認**

```bash
open index.html
```

驗證全功能整合：新增節點 → 設定狀態 → 新增備註 → 跨層拖曳 → 匯出 CSV 含狀態欄位。

- [ ] **Step 7：Commit**

```bash
git add index.html CLAUDE.md
git commit -m "feat: CSV 匯出加入狀態欄；更新 CLAUDE.md 文件"
```
