# Architecture

## 全体構成

単一 HTML ファイル（`training-site.html`）に HTML / CSS / JavaScript を内包。
外部依存は Chart.js（CDN）のみ。ビルドプロセスなし。

```
training-site.html
  ├── <head>  CSS（変数・共通スタイル・タブ・カード・トレーニング画面）
  ├── <body>
  │   ├── #header       共通ヘッダー（タイトル + メインタブ）
  │   ├── #tab-home     ホームセクション
  │   ├── #tab-shun     瞬読セクション（カード一覧 + 各詳細画面）
  │   └── #tab-mem      記憶法セクション（カード一覧 + 各詳細画面）
  └── <script>  JavaScript（状態管理・トレーニングロジック・グラフ・localStorage）
```

## ナビゲーション構造

```
メインタブ（ホーム / 瞬読 / 記憶法）
  └─ 各セクション
       ├── .card-grid        カード一覧（メニュー）
       └── .detail-panel[data-id="xxx"]  詳細画面
             ├── 解説（意義・やり方）
             └── トレーニング本体
```

カード一覧と詳細画面の切り替えは JS で `active` クラスを付け外しし、CSS トランジション（スライドイン）でアニメーションする。

## JavaScript モジュール構成（script 内の論理的区分）

| 区分 | 責務 |
|-----|------|
| タブ・ナビゲーション | メインタブ切替・カード→詳細の開閉 |
| 瞬読トレーニング群 | フラッシュ / 単語 / 円形 / ランダム / 観察力 / イメージ力 の各ロジック |
| 記憶法トレーニング群 | 数字記憶 / 単語記憶 / カード配置記憶 / ドミニクシステム の各ロジック |
| 実力チェックテスト | 瞬読実力テスト・記憶法実力テスト（種目選択）の進行管理 |
| localStorage I/O | `shun_history` / `mem_history` / `dominic_table` の読み書き |
| グラフ描画 | Chart.js を使った履歴グラフ（折れ線・棒） |
| GitHub テキスト連携 | `texts/index.json` fetch → ドロップダウン生成 → `.txt` fetch |
| 内蔵単語リスト | 4〜10字の日常単語（約270語）を配列で保持 |
| 称号ロジック | スコア・到達レベルから称号を判定・付与 |

## データフロー

```
トレーニング完了
  → 結果オブジェクト生成（日時・手法・スコアなど）
  → localStorage[shun_history / mem_history] に push
  → グラフ・称号・スコア表を再描画
```

## localStorage スキーマ

```js
// shun_history / mem_history の各エントリ形式
{
  date: "2026-05-30T10:00:00",
  type: "単語" | "フラッシュ" | "実力テスト(瞬読)" | ...,
  score: number,
  // 手法固有フィールド（speed, accuracy, reachedDigits など）
}

// dominic_table
{
  "00": { person: "", action: "", object: "" },
  "01": { ... },
  // ... 99 まで
}
```

## GitHub テキスト連携

```
起動時
  fetch(`raw.githubusercontent.com/{owner}/{repo}/main/texts/index.json`)
    → ドロップダウンに title 一覧を表示
  ユーザーが選択
    fetch(`raw.githubusercontent.com/{owner}/{repo}/main/{file}`)
      → テキストエリアに展開 → フラッシュ等の題材に使用
  fetch 失敗時
    → 内蔵フォールバックテキストを使用（エラー非表示）
```

## CSS 設計

- CSS カスタムプロパティ（`--color-shun`, `--color-mem` など）でテーマカラーを一元管理
- 瞬読エリアは `--color-shun`（深緑）、記憶法エリアは `--color-mem`（テラコッタ）を適用
- アニメーションは `@keyframes` + `transition` で定義、JS はクラス付け外しのみ行う
- メディアクエリでレスポンシブ対応（PC / タブレット / スマホ）

## トレーニング画面の共通パターン

```
状態変数: isRunning, currentStep, timerId
開始ボタン押下 → isRunning = true → setInterval / setTimeout でステップ進行
停止/完了 → clearInterval → 採点 → localStorage 書き込み → UI 更新
```

各トレーニングはこのパターンに従う。タイマー ID は必ず変数に保持し、画面離脱・再開始時に `clearInterval` して二重起動を防ぐ。
