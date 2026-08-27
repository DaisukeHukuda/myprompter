# myprompter — プレゼン録画用テレプロンプター 設計書

- 日付: 2026-08-27
- 承認: 福田さん（会話内で承認済み）
- 目的: スマホで録画しながら、その後ろに置いたiPad/タブレット/PCに原稿を表示し、上方向に自動スクロールさせて読むためのWebアプリ。

## 目的・成功基準

- **目的**: プレゼン録画時に原稿を自然に読めるプロンプターを用意する。
- **成功基準**:
  - iPadのSafariで開き、原稿が黒背景・白文字で下から上へ自動スクロールする。
  - 録画中に大きなボタンで停止/再開・スピード調整ができる。
  - 原稿・スピード・文字サイズが端末に自動保存され、次回もそのまま使える。
  - 録画中に画面がスリープしない。
- **やらないこと（YAGNI）**:
  - リモコン操作（別端末からの遠隔操作）
  - ミラー表示（左右反転）
  - 複数原稿の保存・切替
  - サーバー・アカウント・課金要素

## 全体方針

- **構成**: 単一HTMLファイル（HTML+CSS+JavaScriptを1ファイルに）。ビルド不要、Vanilla JS。
- **配布**: 開発中はClaude Artifact（非公開URL）→ 後日GitHubをもらってGitHub Pagesへ移行。
- **保存**: localStorage（端末内保存）。読み書きはtry/catchで保護し、使えない環境でも初期原稿で動作する。
- **対応端末**: iPad/タブレット/PC（レスポンシブ）。ダーク背景固定のプロンプター画面のため、テーマ切替は不要。

## 画面構成（2画面）

### ① 編集画面（初期画面）
- 大きな原稿入力欄。初期値として下記「初期原稿」を埋め込む。
- 入力のたびにlocalStorageへ自動保存。保存済み原稿があればそれを優先表示。
- 文字サイズ調整（プロンプター画面のフォントサイズ。設定も自動保存）。
- 「スタート」ボタンでプロンプター画面へ遷移。

### ② プロンプター画面
- 黒背景＋白文字、大きなフォント。下から上への自動スクロール。
- スタート時に「3・2・1」のカウントダウン → スクロール開始。
- **目印ライン**: 画面上端から1/3の位置に薄い横線。ライン付近の行を少し明るくハイライト。
- **残り時間・進捗**: 画面隅に小さく表示（例「残り 2:30 ／ 40%」）。現在のスピードと残りスクロール量から算出。
- **スリープ防止**: Screen Wake Lock APIを使用（Step 3で対応。非対応環境では無視して動作継続）。

## 操作仕様（プロンプター画面）

| 操作 | 方法 |
|---|---|
| 一時停止／再開 | 画面中央の広い領域をタップ |
| スピード調整 | 右端の大きな「＋」「－」ボタン（長押しで連続変化） |
| 少し戻る／進む | 左端の「▲」「▼」ボタン |
| 最初から | 「⟲」ボタンで先頭に戻る（カウントダウンから再開） |
| 編集に戻る | 左上「←」 |
| キーボード（PC） | Space＝停止/再開、↑↓＝スピード、←＝先頭へ |

- ボタンは録画中に少し離れた位置からでも押せるよう大きめ（最低60px四方目安）。
- スピードは段階値。基準（1.0x）は「画面上を1行が約6秒かけて目印ラインを通過する程度」とし、0.5x〜3.0xを0.1刻みで調整可（実機確認で基準値は微調整してよい）。設定は自動保存。

## 技術詳細

- スクロールは `requestAnimationFrame` で毎フレーム位置を更新（滑らかさ優先。CSSアニメーションは途中の速度変更に弱いため不採用）。
- 原稿はプレーンテキストとして扱い、改行・空行を含め入力どおりに表示する（`**` などの記号も特別扱いせずそのまま表示。初期原稿は記号なしで埋め込み済み）。
- 進捗計算: `残り時間 = 残りスクロール距離 ÷ 現在のスクロール速度`。スピード変更時に即再計算。
- localStorageキー: `myprompter.script` / `myprompter.speed` / `myprompter.fontSize`。
- 単一ファイル・外部依存なし（CDN・フォント読込なし。システムフォント使用）。

## エラー処理

- localStorage不可（プライベートモード等）: 保存をスキップし、埋め込み初期原稿で動作。エラー表示はしない。
- Wake Lock非対応: 無視して動作継続（iPad側の自動ロック設定を長めにする案内を編集画面に小さく記載）。
- 原稿が空: プロンプター画面に「原稿を入力してください」を表示し、編集画面へ誘導。

## テスト方針

- 各Stepの完了時にブラウザプレビューで動作確認（スクロール、停止/再開、スピード変更、保存の復元）。
- Step 3でiPad実機（Safari）確認: タップ判定の大きさ、スリープ防止、画面回転。
- ロジックの自動テストは行わない（規模が小さく、UI主体のため手動確認を優先）。

## 開発ステップ

1. **Step 1（MVP）**: 編集画面＋プロンプター画面、自動スクロール、スピード調整、停止/再開、カウントダウン。
2. **Step 2**: 文字サイズ調整、目印ライン、残り時間/進捗、原稿・設定の自動保存。
3. **Step 3**: iPad実機調整（タップ判定、Wake Lock、回転対応）、GitHub Pages移行準備。

各Step完了ごとにArtifactへ公開し、福田さんがiPadで確認 → フィードバック反映 → 次のStepへ。

## 実装体制

- 実装: Sonnetサブエージェント（トークン節約のため）。
- 設計・レビュー・監査: メインセッション（Fable 5）。
- 実装難易度が高い箇所（スクロールエンジン等）が問題を起こした場合のみメインセッションが直接修正。

## 初期原稿（埋め込み用・プレーンテキスト）

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
