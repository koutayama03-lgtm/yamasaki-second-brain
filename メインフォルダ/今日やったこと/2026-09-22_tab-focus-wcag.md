---
tags: [java, javascript, html, accessibility, wcag, tabindex, jquery-ui, debug]
date: 2026-09-22
---

# 2026-09-22 Tabキーフォーカス遷移の理解を深める（続き）

前回（[[2026-09-16_tab-focus-bug]]）の続き。今日は理論的背景の整理と、不具合原因の深掘りをした。

## 1. 理想的なフォーカス遷移とその根拠

- 理想は視覚的な読み順（Z字型：左→右、上→下の繰り返し）にTab順を一致させること
- 根拠：WCAG 達成基準 2.4.3「フォーカス順序」（意味と操作性を保つ順序でフォーカスを受け取れるようにする）
- 技術的な裏付け：ブラウザはHTMLの**ソースコード順（DOM順）**をそのままTab順として使う
  → マークアップの記述順と見た目の並びさえ一致していれば、特別なことをしなくても自然と理想形になる
- `tabindex`に正の数値を振って無理やり順序を制御するのはアンチパターン
  → WCAGの補足技術文書「F44」で名指しされている失敗パターンそのもの

## 2. 不具合の根本原因調査

- 不具合の実体：`tabindex`の正の値が振られている要素と、振られていない要素が混在し、WCAGに沿わないフォーカス順になっている
- `git blame`で確認 → 最初のコミットから既に`tabindex`の正の値が入っていた
- 社内の開発企画書に「画面上のオブジェクトには必ずtabindexの値を明示的に示すこと」という規約が存在すると判明
  → WCAGの推奨（F44）と真逆の内容を、社内標準として明文化してしまっている状態

### 原因の仮説：RAD系GUIフレームワークの開発文化の名残

- 会社は以前**Delphi**（Borland/Embarcadero製、Object Pascalベースの伝統的なRAD系IDE）で開発していたらしい
- Delphiのような RAD系GUIフレームワーク（VB6, PowerBuilder, WinForms, ASP.NET WebForms, Java Swing/AWT等も同系統）は、コントロールをキャンバス上に自由な座標で配置する方式
  → HTMLの「ソース記述順」に相当する暗黙の順序ルールが存在しない
  → だからこそ各コントロールに`TabOrder`（Delphi）/`TabIndex`（VB6・WebForms）を**明示的に指定するのが正しい作法**だった
- Delphiの`.dfm`ファイルには、各部品の座標とは別に`TabOrder`プロパティが記録される。IDEにも専用のタブオーダー編集ダイアログがあった
- この「明示するのが当たり前」という開発文化が、Web開発へ移行した際にそのまま企画書に持ち込まれてしまった可能性が高い
  ※ あくまで状況証拠からの推測。断定するには企画書の制定時期など裏付けが必要

## 3. jQuery UIとの関係

- 該当画面では jQuery UI Accordion を使用している
- ブラウザのTab順アルゴリズム：**正のtabindexを持つ要素が数値の小さい順に最優先で処理され、その後にtabindex=0または未指定の要素がDOM順で処理される**という二段構え
- jQuery UI Accordionは「ローミングTabindex」というWAI-ARIAの正しい設計パターンを使用
  - アクティブなヘッダー → `tabindex="0"`
  - 非アクティブなヘッダー → `tabindex="-1"`（矢印キーでの移動やクリックは可能。Tab巡回からのみ除外）
  - これ自体は正しい実装
- 問題は、周囲の要素に振られた正のtabindexが常に優先されるため、正しく実装されているはずのアコーディオンのヘッダーが、想定と違う順番（多くの場合かなり後方）に押し出されてしまうこと
- これはAccordion固有の話ではなく、jQuery UI全体（Tabs, Menu, Selectmenu, Slider, Datepicker, Dialog等）に共通する設計
- 導入方法の確認：`import`文ではなく`<script>`タグで読み込むのが一般的（jQuery本体→jQuery UIの順で読み込む必要あり）。CSSテーマも別途`<link rel="stylesheet">`で読み込む

## 4. フォーカス関連属性の整理

| 属性 | Tab順からの除外 | フォーカス自体の可否 | クリック操作 | 対象要素 |
|---|---|---|---|---|
| 何もなし（`a[href]`/`button`/`input`/`select`/`textarea`等） | ✕（暗黙にtabindex=0相当） | 可 | 可 | ネイティブ対話要素 |
| `tabindex="-1"` | ○（Tab巡回から除外） | 可（`.focus()`やクリックは有効） | 可 | どの要素にも指定可 |
| `disabled` | ○ | 不可（`.focus()`も無視される） | 不可 | button/input/select/textarea/fieldsetのみ。**aタグには無効**（無視される） |
| `readonly` | ✕（Tab順・フォーカスには無関係） | 可 | 可（選択・コピーは可、編集不可） | input(一部type)・textareaのみ。**disabledと違いフォーム送信時に値が送信される** |

- 暗黙のtabindexを持つ要素（何も書かなくてもフォーカス対象）：`a[href]`, `button`, `input`（type=hidden除く）, `select`, `textarea`, `iframe`, `audio/video[controls]`, `contenteditable`要素
- 上記以外（`div`, `span`, `li`等）は`tabindex`を明示しない限りフォーカス対象外
- 「押させたくない」の意味で使い分けが必要
  - 完全に操作不可にしたい → `disabled`（対応要素のみ）or `aria-disabled` + JS制御（`a`など非対応要素）
  - Tab巡回だけ外したいが操作自体は許可 → `tabindex="-1"`（jQuery UIのローミングTabindexがまさにこれ）

## 5. ブラウザ自体のUI（Chromeメニュー・ブックマークバー等）との境界線

- ブックマークバーやハンバーガーメニューは、Webページの**DOMには属さない**
- Chromeは独自の「Views」というUIツールキットで描画しており、`FocusManager`という独自の仕組みでTab順を管理
- OSのネイティブなウィンドウ・フォーカスシステム（Win32/Cocoa等）の上に構築されている、ブラウザベンダー独自の実装
- Web標準（HTML/WCAG）とは無関係なため、デベロッパーツールでは一切見えない
- 自分のページでコンテンツを全て辿り終えてからChromeメニューに抜けるのは正常な標準挙動

## 6. 実務上の疑問：同僚の画面との挙動差（未検証・仮説段階）

- 自分の画面：コンテンツ内のフォーカス遷移が全て終わってからChromeメニューへ抜ける（正常挙動）
- 同僚の画面：早い段階でChromeメニューに飛ぶことがあるらしい
- 仮説
  1. その画面はそもそもフォーカス可能な要素の絶対数が少ない（閲覧専用に近い画面等）
  2. 要素は存在するが `display:none` / `disabled` / `tabindex="-1"` 等でフォーカス対象から除外されている
  3. カスタムJS（keydownイベントの横取り等）にバグがあり、`preventDefault()`漏れ等で本来のTab移動処理を妨げている
- 要検証：デベロッパーツールのAccessibilityタブ、または実機でのTab移動確認

## 次にやること

- 実際のコードで、周囲の要素の`tabindex`値とjQuery UI Accordionの`tabindex="0"/"-1"`の値を突き合わせて確認する
- 同僚の画面のDOM構造を確認し、フォーカス可能要素の数・状態（display / disabled / tabindex）を調べる
- 上記の裏付けが取れたら、企画書（「全オブジェクトに明示的にtabindexを付与」の規約）の改訂を提案する準備をする
