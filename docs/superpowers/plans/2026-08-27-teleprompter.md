# テレプロンプターWebアプリ 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** プレゼン録画時にiPad等で原稿を自動スクロール表示する単一HTMLのテレプロンプターを作る。

**Architecture:** `index.html` 1ファイルにHTML/CSS/JSをすべて内包。画面は編集画面とプロンプター画面の2つで、`hidden`クラスの付け外しで切替。スクロールは`requestAnimationFrame`で毎フレーム`transform: translateY()`を更新する方式。保存はlocalStorage（try/catch保護）。

**Tech Stack:** Vanilla JS / CSS（フレームワーク・ビルドツール・外部依存なし。CDN読込禁止、システムフォントのみ）

**Spec:** `docs/superpowers/specs/2026-08-27-teleprompter-design.md`（必ず読むこと）

## Global Constraints

- 成果物は `index.html` 1ファイルのみ（テスト・ビルド設定・依存ファイルを作らない）
- 外部ネットワーク読込なし（CDN・Webフォント・画像URL禁止）
- 自動テストは書かない（specの決定事項。各タスクはブラウザでの手動確認で完了とする）
- プロンプター画面は黒背景 `#000` ＋ 白文字 `#fff` 固定。テーマ切替なし
- UIの文言は日本語（原稿本文は入力されたまま表示）
- localStorageキーは `myprompter.script` / `myprompter.speed` / `myprompter.fontSize` のみ
- コミットメッセージは日本語で `feat:` / `fix:` プレフィックス
- 動作確認は `python3 -m http.server 8080` を起動し、Browserペイン（`preview_start` → `http://localhost:8080`）で行う。確認後スクリーンショットで目視検証すること

## DOM ID・状態・関数の共通定義（全タスク共通の契約）

```
画面:      #edit-screen, #prompter-screen  (非表示側に class="hidden")
編集画面:  #script-input (textarea), #font-size-input (range), #font-size-value (span),
           #start-btn (button)
プロンプター: #scroll-container (viewport), #script-content (translateYで動く内側),
           #countdown-overlay, #marker-line, #progress-display,
           #pause-zone (中央タップ領域), #speed-up, #speed-down, #speed-value,
           #nudge-back, #nudge-forward, #restart-btn, #back-btn
JS状態:    state = { speed: 1.0, fontSize: 48, scrollY: 0, playing: false }
主要関数:  showEdit(), showPrompter(), startCountdown(), play(), pause(), togglePlay(),
           tick(timestamp), pxPerSec(), renderScript(text), updateProgress(),
           updateMarkerHighlight(), saveSettings(), loadSettings(), nudge(lines),
           restart()
速度計算:  pxPerSec() = (state.fontSize * 1.6 / 6) * state.speed
           （1.0x = 1行(行高=fontSize*1.6px)が約6秒で通過する速さ）
```

---

### Task 1: 画面骨格と2画面切替（編集⇄プロンプター）＋カウントダウン

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: 上記共通定義のDOM ID一式、`showEdit()` / `showPrompter()` / `startCountdown()` / `renderScript(text)`、定数 `DEFAULT_SCRIPT`

- [ ] **Step 1: index.htmlを作成（骨格＋編集画面＋プロンプター画面の器＋カウントダウン）**

以下の要件をすべて満たすこと：

1. `<!DOCTYPE html>`、`lang="ja"`、`<meta name="viewport" content="width=device-width, initial-scale=1">`、`<title>myprompter</title>`
2. 編集画面 `#edit-screen`：
   - タイトル「myprompter」
   - `#script-input`（textarea、画面の大半を占める。初期値は後述の `DEFAULT_SCRIPT`）
   - `#start-btn`「スタート ▶」（高さ60px以上の大ボタン）
   - 下部に小さな注意書き「録画中に画面が消える場合は、端末の自動ロック設定を長めにしてください」
3. プロンプター画面 `#prompter-screen`（初期状態 `class="hidden"`）：黒背景全面。内部に `#scroll-container`（`overflow: hidden; height: 100%`）と、その中の `#script-content`。原稿は `renderScript(text)` で1行＝1つの `<div class="line">` として描画（空行は `<div class="line blank">&nbsp;</div>`）。文字色 `#fff`、`font-size: 48px`、`line-height: 1.6`、左右パディング8%。**原稿の先頭に画面高さ40%分・末尾に画面高さ70%分の余白**（`padding-top: 40vh; padding-bottom: 70vh` を `#script-content` に）を入れ、最初の行が下方から現れ、最後の行が読み切れるようにする
4. `#countdown-overlay`：全面オーバーレイに「3」→「2」→「1」を各1秒、`font-size: 30vh` 中央表示。終了後に非表示
5. `#back-btn`「←」を左上に配置。押すと `showEdit()`
6. JS：`const DEFAULT_SCRIPT = \`...\`` に下記原稿を**一字一句そのまま**埋め込む：

```
Hi, I'm Daisuke.
Welcome to Oku-Nikko.

I'm the founder of Sup! Sup!,
and I guide guests here in both English and Chinese.

One of my favorite ways to show you this area
is Paddle Boarding on Lake Chuzenji.

For me, Paddle Boarding is not just about being on the water.
It gives you a chance to slow down, look around,
and really feel the nature of Oku-Nikko.

While we paddle, I love sharing stories about
how this landscape was shaped by volcanic activity,
Also the local history and culture.

My goal is to help you enjoy the scenery,
but also understand why this place is so special.

I hope you leave Oku-Nikko
feeling a real connection to this place.
```

7. `#start-btn` → `showPrompter()`：textareaの内容を `renderScript()` で描画し、プロンプター画面を表示して `startCountdown()` を呼ぶ。原稿が空白のみなら描画の代わりに「原稿を入力してください」を中央表示し、カウントダウンしない
8. `startCountdown()` はこの時点では「カウントダウン表示が終わったら `#countdown-overlay` を隠す」まで（スクロール開始はTask 2で接続）

- [ ] **Step 2: ブラウザで動作確認**

`python3 -m http.server 8080` を起動し、Browserペインで `http://localhost:8080` を開いて確認：
- 編集画面に初期原稿が入っている
- スタート → 黒画面に 3→2→1 → 原稿が白文字で表示されている（まだ動かない）
- 「←」で編集画面に戻れる
- 原稿を全部消してスタート →「原稿を入力してください」

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: 画面骨格と編集⇄プロンプター切替、カウントダウンを実装"
```

---

### Task 2: スクロールエンジン（自動スクロール・停止/再開・スピード変更）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 1の `startCountdown()`, `#script-content`, `state`
- Produces: `play()`, `pause()`, `togglePlay()`, `tick(timestamp)`, `pxPerSec()`, `setSpeed(v)`, `restart()`, `nudge(lines)`

- [ ] **Step 1: rAFスクロールエンジンを実装**

核となるロジック（このまま使ってよい）：

```js
let lastTs = null;
function pxPerSec() { return (state.fontSize * 1.6 / 6) * state.speed; }
function maxScroll() {
  return Math.max(0, scriptContent.scrollHeight - scrollContainer.clientHeight);
}
function tick(ts) {
  if (!state.playing) return;
  if (lastTs !== null) {
    state.scrollY = Math.min(state.scrollY + pxPerSec() * (ts - lastTs) / 1000, maxScroll());
    scriptContent.style.transform = `translateY(${-state.scrollY}px)`;
    if (state.scrollY >= maxScroll()) { pause(); }  // 末尾到達で自動停止
  }
  lastTs = ts;
  requestAnimationFrame(tick);
}
function play()  { if (state.playing) return; state.playing = true; lastTs = null; requestAnimationFrame(tick); }
function pause() { state.playing = false; }
function togglePlay() { state.playing ? pause() : play(); }
function setSpeed(v) { state.speed = Math.min(3.0, Math.max(0.5, Math.round(v * 10) / 10)); }
function nudge(lines) {  // 戻る=負、進む=正。1行 = fontSize*1.6px
  state.scrollY = Math.min(maxScroll(), Math.max(0, state.scrollY + lines * state.fontSize * 1.6));
  scriptContent.style.transform = `translateY(${-state.scrollY}px)`;
}
function restart() { pause(); state.scrollY = 0; scriptContent.style.transform = 'translateY(0)'; startCountdown(); }
```

接続：`startCountdown()` の「1」表示後にオーバーレイを隠して `play()`。`showPrompter()` 冒頭で `state.scrollY = 0` にリセット。`showEdit()` で `pause()`。

- [ ] **Step 2: 一時停止の中央タップ領域とキーボード操作を実装**

- `#pause-zone`：プロンプター画面中央の透明領域（幅・高さとも画面の約60%）。クリック/タップで `togglePlay()`。一時停止中は画面隅に小さく「⏸ 一時停止中（タップで再開）」を表示
- キーボード：プロンプター画面表示中のみ有効。`Space`=togglePlay（`preventDefault`でページスクロール防止）、`ArrowUp`=setSpeed(+0.1)、`ArrowDown`=setSpeed(-0.1)、`ArrowLeft`=restart()

- [ ] **Step 3: ブラウザで動作確認**

- スタート → カウントダウン後、文字が上方向へ滑らかに流れる
- 中央タップで停止・再開できる。スペースキーでも同様
- ↑↓でスピードが変わる（体感で確認）
- 最後まで流れたら自動停止する

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat: 自動スクロールエンジンと停止/再開・スピード変更を実装"
```

---

### Task 3: 画面上の操作ボタン（スピード±・戻る/進む・最初から）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 2の `setSpeed()`, `nudge()`, `restart()`, `state.speed`
- Produces: `#speed-up` / `#speed-down` / `#speed-value` / `#nudge-back` / `#nudge-forward` / `#restart-btn` のUI一式

- [ ] **Step 1: ボタンUIを実装**

- 右端縦並び：`#speed-up`「＋」、`#speed-value`（現在値「1.0x」表示）、`#speed-down`「－」。タップで±0.1。**長押しで連続変化**（`pointerdown`で250ms後から100ms間隔のsetInterval、`pointerup`/`pointerleave`で解除）
- 左端縦並び：`#nudge-back`「▲」= `nudge(-3)`（3行戻る）、`#nudge-forward`「▼」= `nudge(3)`
- 左上：`#back-btn`「←」の下に `#restart-btn`「⟲」= `restart()`
- 全ボタン共通スタイル：60px四方以上、半透明の暗いグレー地（`rgba(255,255,255,0.12)`）に白文字、角丸、`user-select: none`、`touch-action: manipulation`。ボタン類のタップが `#pause-zone` に伝播しないこと（配置を重ねない or `stopPropagation`）
- スピード変更時は `#speed-value` を即時更新

- [ ] **Step 2: ブラウザで動作確認**

- ＋／－でスピード表示が0.1刻みで変わり、0.5x〜3.0xでちゃんと止まる
- ＋の長押しで連続的に上がる
- ▲で戻り、▼で進む（スクロール中でも位置が飛ぶ）
- ⟲でカウントダウンからやり直しになる
- ボタンを押しても再生/停止が切り替わらない（誤爆しない）

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: スピード±・戻る/進む・最初からのボタンを実装"
```

---

### Task 4: 目印ライン・残り時間/進捗・文字サイズ調整

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: Task 2の `tick()`, `pxPerSec()`, `maxScroll()`, `state.fontSize`
- Produces: `updateProgress()`, `updateMarkerHighlight()`, `#marker-line`, `#progress-display`, `#font-size-input`

- [ ] **Step 1: 目印ラインとハイライトを実装**

- `#marker-line`：プロンプター画面の上から33%の位置に固定表示する横線（高さ2px、`rgba(255,255,255,0.35)`、左右いっぱい。`pointer-events: none`）
- `updateMarkerHighlight()`：`tick()` 内および `nudge()`/`restart()` 後に呼ぶ。マーカーのY座標（`scrollContainer.clientHeight * 0.33`）に重なる `.line` 要素を特定し、その行だけ `class="current"` を付与（`document.elementFromPoint` は使わず、`scrollY` と各行の `offsetTop` から計算）。CSS：通常行は `opacity: 0.75`、`.current` は `opacity: 1` ＋ `font-weight: 600`

- [ ] **Step 2: 残り時間・進捗表示を実装**

- `#progress-display`：右上に小さく「残り 2:30 ／ 40%」形式で表示（`font-size: 14px`、`opacity: 0.6`）
- `updateProgress()`：`残り秒 = (maxScroll() - state.scrollY) / pxPerSec()`、`進捗% = maxScroll()が0なら100、それ以外は round(scrollY / maxScroll() * 100)`。`分:秒` は `Math.floor(s/60) + ':' + String(Math.floor(s%60)).padStart(2,'0')`。`tick()` 内では毎フレームでなく約500ms間隔で更新。スピード変更時は即時更新

- [ ] **Step 3: 文字サイズ調整を実装**

- 編集画面に `#font-size-input`（`type="range" min="24" max="96" step="4"`、初期値48）と `#font-size-value`「48px」を追加。ラベル「文字サイズ」
- 変更で `state.fontSize` を更新し `#script-content` の `font-size` に反映（プロンプター画面へ行ったとき反映されていればよい）

- [ ] **Step 4: ブラウザで動作確認**

- マーカー線が画面上部1/3に見え、通過中の行だけ明るく太くなる
- 右上に残り時間と%が出て、スクロールに応じて減っていく。スピードを上げると残り時間が短くなる
- 文字サイズを96にしてスタートすると大きな文字で表示される

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: 目印ライン・残り時間/進捗・文字サイズ調整を実装"
```

---

### Task 5: 自動保存（原稿・スピード・文字サイズ）

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `state`, `#script-input`, `#font-size-input`, `setSpeed()`
- Produces: `saveSettings()`, `loadSettings()`, `saveScript()`

- [ ] **Step 1: localStorage保存を実装**

```js
const LS = { script: 'myprompter.script', speed: 'myprompter.speed', fontSize: 'myprompter.fontSize' };
function lsGet(k) { try { return localStorage.getItem(k); } catch (e) { return null; } }
function lsSet(k, v) { try { localStorage.setItem(k, v); } catch (e) { /* 保存不可環境では黙ってスキップ */ } }
function saveScript() { lsSet(LS.script, scriptInput.value); }
function saveSettings() { lsSet(LS.speed, String(state.speed)); lsSet(LS.fontSize, String(state.fontSize)); }
function loadSettings() {
  const script = lsGet(LS.script);
  scriptInput.value = (script !== null && script !== '') ? script : DEFAULT_SCRIPT;
  const speed = parseFloat(lsGet(LS.speed));
  if (!isNaN(speed)) state.speed = Math.min(3.0, Math.max(0.5, speed));
  const fs = parseInt(lsGet(LS.fontSize), 10);
  if (!isNaN(fs)) state.fontSize = Math.min(96, Math.max(24, fs));
}
```

- 起動時に `loadSettings()` を呼び、textarea・スライダー・スピード表示に反映
- `#script-input` の `input` イベントで `saveScript()`（デバウンス300ms）
- スピード変更・文字サイズ変更のたびに `saveSettings()`

- [ ] **Step 2: ブラウザで動作確認**

- 原稿の一部を書き換え → リロード → 書き換えが残っている
- スピードを1.5x、文字サイズを64にする → リロード → 値が復元されている
- localStorageをクリア（DevToolsで `localStorage.clear()`）→ リロード → 初期原稿と初期値に戻る

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: 原稿・スピード・文字サイズの自動保存を実装"
```

---

### Task 6: スリープ防止（Wake Lock）とタッチ環境の仕上げ

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `showPrompter()`, `showEdit()`
- Produces: `acquireWakeLock()`, `releaseWakeLock()`

- [ ] **Step 1: Wake Lockを実装**

```js
let wakeLock = null;
async function acquireWakeLock() {
  try {
    if ('wakeLock' in navigator) wakeLock = await navigator.wakeLock.request('screen');
  } catch (e) { /* 非対応・拒否時は何もしない */ }
}
function releaseWakeLock() { if (wakeLock) { wakeLock.release().catch(() => {}); wakeLock = null; } }
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible' && !prompterScreen.classList.contains('hidden')) acquireWakeLock();
});
```

- `showPrompter()` で `acquireWakeLock()`、`showEdit()` で `releaseWakeLock()`

- [ ] **Step 2: タッチ環境の仕上げ**

- プロンプター画面表示中のピンチズーム・ダブルタップズーム・選択・スクロールバウンスを防止：`#prompter-screen` に `touch-action: none`（ボタンは `manipulation` のまま）、`body` に `overscroll-behavior: none`、プロンプター画面の全要素に `-webkit-user-select: none`
- `dblclick` の `preventDefault`
- 画面回転・リサイズ時（`resize` イベント）：`maxScroll()` 前提が変わるため `state.scrollY` を `maxScroll()` 内にクランプし、マーカーハイライトと進捗を再計算

- [ ] **Step 3: ブラウザで動作確認**

- Browserペインをモバイルサイズ（`resize_window` でtablet/mobile）にして表示崩れがないこと（ボタンが重ならない、原稿がはみ出さない）
- リサイズしてもスクロール位置が壊れない
- コンソールにエラーが出ていない（`read_console_messages` で確認）

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat: Wake Lockによるスリープ防止とタッチ操作の仕上げ"
```

---

## タスク完了後の全体チェックポイント（メインセッションが実施）

1. 全機能の通し確認（ブラウザ＋モバイルエミュレーション）
2. Artifactへ公開し、福田さんにiPad実機確認を依頼
3. フィードバック反映後、GitHub（origin/main）へpush。GitHub Pages有効化を案内
