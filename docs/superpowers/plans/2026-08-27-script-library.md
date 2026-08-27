# 原稿リスト（保存・履歴）機能 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 複数の原稿をlocalStorageに保存し、編集画面のリストから切替できるようにする（更新順リストが履歴を兼ねる）。

**Architecture:** 既存の単一 `index.html` に、原稿配列（`myprompter.docs`）と選択中ID（`myprompter.currentDocId`）のデータ層を追加。既存の `saveScript()` を「選択中原稿への書き込み」に置き換え、編集画面にセレクト＋新規＋削除のUIを1行追加する。プロンプター画面は変更しない。

**Tech Stack:** Vanilla JS / localStorage（外部依存なし、現行構成のまま）

**Spec:** `docs/superpowers/specs/2026-08-27-script-library-design.md`

## Global Constraints

- 成果物は `index.html` 1ファイルのみ。外部ネットワーク読込なし。自動テストなし（ブラウザ手動確認）
- localStorage新キーは `myprompter.docs` / `myprompter.currentDocId` の2つのみ。`myprompter.script` は移行後に削除
- 保存失敗は黙ってスキップ（既存の lsGet/lsSet を使う）
- UIの文言は日本語。コミットは `feat:`/`fix:` プレフィックス（日本語）

---

### Task 1: データ層（原稿配列・移行・起動配線）

**Files:**
- Modify: `index.html`（LS定義、saveScript/loadSettings周辺）

**Interfaces:**
- Produces: `docs`（配列）, `currentDocId`, `genId()`, `currentDoc()`, `sortedDocs()`, `docTitle(doc)`, `loadDocs()`, `saveDocs()`, `flushScriptSave()`。Task 2のUIはこれらを呼ぶ

- [ ] **Step 1: LSキー追加とデータ層実装**

`LS` 定義に `docs: 'myprompter.docs', currentDocId: 'myprompter.currentDocId'` を追加し、以下を実装:

```js
let docs = [];          // [{id, text, updatedAt}]
let currentDocId = null;

function genId() { return Date.now().toString(36) + Math.random().toString(36).slice(2, 6); }
function currentDoc() { return docs.find(function (d) { return d.id === currentDocId; }); }
function sortedDocs() { return docs.slice().sort(function (a, b) { return b.updatedAt - a.updatedAt; }); }
function docTitle(doc) {
  const first = (doc.text.split('\n').find(function (l) { return l.trim() !== ''; }) || '').trim();
  const name = first === '' ? '（無題）' : first.slice(0, 20);
  const d = new Date(doc.updatedAt);
  return name + ' — ' + (d.getMonth() + 1) + '/' + d.getDate();
}
function saveDocs() { lsSet(LS.docs, JSON.stringify(docs)); lsSet(LS.currentDocId, currentDocId); }
function loadDocs() {
  let parsed = null;
  try { parsed = JSON.parse(lsGet(LS.docs)); } catch (e) { parsed = null; }
  if (Array.isArray(parsed) && parsed.length > 0 &&
      parsed.every(function (d) { return d && typeof d.id === 'string' && typeof d.text === 'string' && typeof d.updatedAt === 'number'; })) {
    docs = parsed;
  } else {
    const legacy = lsGet(LS.script);  // 旧単一原稿からの移行
    docs = [{ id: genId(), text: (legacy !== null && legacy !== '') ? legacy : DEFAULT_SCRIPT, updatedAt: Date.now() }];
    try { localStorage.removeItem(LS.script); } catch (e) {}
  }
  currentDocId = lsGet(LS.currentDocId);
  if (!currentDoc()) currentDocId = sortedDocs()[0].id;
  saveDocs();
}
```

- [ ] **Step 2: saveScript/デバウンス/起動処理を置き換え**

```js
function saveScript() {
  const doc = currentDoc();
  if (!doc) return;
  doc.text = scriptInput.value;
  doc.updatedAt = Date.now();
  saveDocs();
  renderDocSelect();  // タイトル・並び順が変わるため（Task 2で定義。Task 1の時点では空関数を置く）
}
function flushScriptSave() {  // 切替・新規・削除の直前に、保留中の編集を確実に書き込む
  if (scriptSaveTimer !== null) { clearTimeout(scriptSaveTimer); scriptSaveTimer = null; saveScript(); }
}
```

- 既存の input デバウンスは `scriptSaveTimer = setTimeout(function () { scriptSaveTimer = null; saveScript(); }, 300);` の形に変更（タイマー保留中かを判定できるように）
- `loadSettings()` から原稿読み込み部分（`LS.script` 参照）を削除し speed/fontSize のみ残す
- 起動時: `loadSettings(); loadDocs(); scriptInput.value = currentDoc().text; renderDocSelect(); ...`

- [ ] **Step 3: ブラウザ確認（移行と復元）**

`python3 -m http.server 8090` → Browserペインで確認:
- 旧データ移行: `localStorage.clear(); localStorage.setItem('myprompter.script','テスト原稿');` → リロード → textareaに「テスト原稿」、`myprompter.docs` に1件、`myprompter.script` は削除済み
- `localStorage.clear()` → リロード → DEFAULT_SCRIPT 1件で起動
- 編集 → リロード → 内容が復元される

- [ ] **Step 4: コミット**

```bash
git add index.html && git commit -m "feat: 原稿リストのデータ層（複数原稿・移行・自動保存）を実装"
```

---

### Task 2: 原稿リストUI（セレクト・新規・削除）

**Files:**
- Modify: `index.html`（編集画面マークアップ、CSS、イベント配線）

**Interfaces:**
- Consumes: Task 1 の `docs/currentDocId/currentDoc()/sortedDocs()/docTitle()/saveDocs()/flushScriptSave()/genId()`
- Produces: `#doc-select`, `#doc-new`, `#doc-delete`, `renderDocSelect()`, `switchDoc(id)`

- [ ] **Step 1: マークアップとCSS**

タイトル `<h1>` の直後に追加:

```html
<div id="doc-row">
  <select id="doc-select"></select>
  <button id="doc-new">＋ 新規</button>
  <button id="doc-delete">🗑 削除</button>
</div>
```

```css
#doc-row { display: flex; gap: 8px; margin-bottom: 12px; }
#doc-row select { flex: 1; min-height: 44px; font-size: 16px; min-width: 0; }
#doc-row button { min-height: 44px; padding: 0 14px; font-size: 15px; white-space: nowrap; }
```

- [ ] **Step 2: renderDocSelect / switchDoc / 新規 / 削除**

```js
function renderDocSelect() {
  docSelect.innerHTML = '';
  sortedDocs().forEach(function (doc) {
    const opt = document.createElement('option');
    opt.value = doc.id;
    opt.textContent = docTitle(doc);
    docSelect.appendChild(opt);
  });
  docSelect.value = currentDocId;
}
function switchDoc(id) {
  flushScriptSave();
  currentDocId = id;
  scriptInput.value = currentDoc().text;
  saveDocs();
  renderDocSelect();
}
docSelect.addEventListener('change', function () { switchDoc(docSelect.value); });
docNewBtn.addEventListener('click', function () {
  flushScriptSave();
  const doc = { id: genId(), text: '', updatedAt: Date.now() };
  docs.push(doc);
  currentDocId = doc.id;
  scriptInput.value = '';
  saveDocs();
  renderDocSelect();
  scriptInput.focus();
});
docDeleteBtn.addEventListener('click', function () {
  const doc = currentDoc();
  if (!doc) return;
  if (!confirm('「' + docTitle(doc) + '」を削除しますか？')) return;
  docs = docs.filter(function (d) { return d.id !== doc.id; });
  if (docs.length === 0) docs.push({ id: genId(), text: '', updatedAt: Date.now() });
  currentDocId = sortedDocs()[0].id;
  scriptInput.value = currentDoc().text;
  saveDocs();
  renderDocSelect();
});
```

（要素バインド `docSelect/docNewBtn/docDeleteBtn` は既存の const 群に追加。Task 1で置いた空の `renderDocSelect` を本実装に差し替え）

- [ ] **Step 3: ブラウザ確認**

- 新規 → 入力 → セレクトで元の原稿に戻す → 再度切替 → 双方の内容が保持されている
- セレクトの表示が「1行目 — M/D」で、編集した原稿が先頭に来る
- 削除（確認ダイアログ → リスト先頭に切替）、最後の1件削除 → 空の新規原稿
- リロードで選択中原稿・リストが復元
- スタート → プロンプター画面が従来どおり動く（回帰確認）
- コンソールエラーなし

- [ ] **Step 4: コミット・プッシュ・デプロイ確認**

```bash
git add index.html && git commit -m "feat: 原稿リストUI（切替・新規・削除）を追加" && git push origin main
# https://daisukehukuda.github.io/myprompter/ に反映されるまでポーリング（doc-select を grep）
```
