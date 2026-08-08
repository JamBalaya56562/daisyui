# daisyUI オープンIssue 調査レポート

> 対象リポジトリ: [`saadeghi/daisyui`](https://github.com/saadeghi/daisyui) / 更新日: 2026-08-06  
> 対象: 現在オープンな **Issue 全15件**（`master` @ `915d8fd`、v5.7.16）  
> 全件を **「今なら実装できないか」** の観点で調査（実ブラウザ計測 Chromium 147/148/151・Firefox 150、上流バグトラッカー・ブラウザ対応状況の確認を含む）。過去の「要設計」判定の多くは**当時のスナップショットに過ぎず、現在は実装可能**と判明した。

## サマリー

| 分類 | 件数 | 対応方針 |
|---|---:|---|
| 🚀 **PR提出済み・レビュー待ち** | 1 | #4488（[PR #4641](https://github.com/saadeghi/daisyui/pull/4641)） |
| ✅ **実装推奨**（原因・修正方針が確定・実測検証済み） | 4 | #4534／**#4610**（#4534と同一パッチで解消）／#3568／#4436 |
| 📝 **ドキュメント修正**（軽微・即対応可） | 4 | **#4658**／**#4648**／#4616(docs半分)／#4138(docs 1行) |
| 🎨 **要設計判断（真に保留）** | 3 | #3804／#4310／#4465 |
| ⛔ **既存オープンPRあり（引き取り不可）** | 2 | #4393／#4513 |
| 🟣 **上流で解決済み（対応不要）** | 1 | #4549（Chromium M151・stable 2026-07-28／2026-08-06 に実機再確認済み） |
| **合計** | **15** | |

> **調査で判明した主な訂正**
> - **#4488** — 旧「🚫再現不能」は**誤り**。Vite 8 の経路（lightningcss + targets）で決定的に再現し、**1行のガードで修正・検証済み**（PR提出済み）。
> - **#4549** — Chromium 側で**修正済み**（M151、stable 2026-07-28）。daisyUI のコード変更は不要。**2026-08-06 に Chrome 151.0.7922.34 で再測定し解消を確認**。
> - **#4534** — 従来の候補 `contain: layout` は実測で**却下が正解**（Firefox で効果ゼロ・Chrome で視覚的に切れる）。代わりに anchor positioning が**全エンジンで利用可能**（Firefox 147 で出揃い済み）。
> - **#3804** — 論点が違っていた。実測で `appearance: base-select` は**無罪**、真因は `:has()` ルール群。
> - **#4393 / #4513** — 「停滞＝引き取れる」は**誤り**。両PRとも `MERGEABLE` で生きており、待っているのはメンテナのレビュー（2026-08-06 再確認）。
> - **#4658** — 報告者の理由（「今の Ember に index.html が無い」）は**事実誤認**。ただし該当手順は別の理由で**実際に誤り**（下記）。

---

## 🚀 PR提出済み・レビュー待ち（1件）

### [#4488](https://github.com/saadeghi/daisyui/issues/4488) — Vite 8 の本番ビルドでカスタムテーマ色が組込みテーマに上書きされる

> ✅ **[PR #4641](https://github.com/saadeghi/daisyui/pull/4641) 提出済み**（`MERGEABLE`・CI SUCCESS・レビュー待ち）。こちらから追加でやることはない。

- **分類**: `js-behavior-bug`（CSS生成） / **判定**: `valid`（確信度 `high`・実再現） / **深刻度**: `high` / **難易度**: `trivial`
- **根本原因**: `packages/daisyui/functions/pluginOptionsHandler.js` の「other themes」ループ（88-92行）が、直前の `--default` ループ（68-74行）で**既に出力済みのテーマを二重出力**する。宣言内容が同一でセレクタリストが部分集合という重複ペアになるため、**lightningcss（Vite 8 が `build.target` から targets を渡す経路）がルールをマージして `:is()` に畳み**、詳細度が (0,1,0) → **(0,4,1) に跳ね上がる**。結果 `[data-theme="light"]` の要素で組込みテーマがカスタム定義に勝つ。`themes: "all"` 経路（42-57行）にも同型の重複がある。
- **再現条件との一致**: `:where(:root)`（属性なしの初期表示）ではカスタム色が残り、`[data-theme]` 付与後に組込みへ戻る＝報告者の「初回は正しいが切替後に戻る」と完全一致。
- **修正方針**: 「other themes」ループで `--default` 済みテーマをスキップするガードを追加（`if (flags.includes("--default")) return`）。`themes:"all"` 側にも同等のガード。**適用後に lightningcss(minify+targets) 通過で正しい出力になることを確認済み**（副次的に未圧縮CSSが約1.8KB減）。
- **根拠**: tailwindcss 4.3.1 CLI + lightningcss 1.32.0 で再現→修正→再検証。なお pdanpdan のコメント（2026-04-03）「doctype に data-theme を書いているのが原因」は誤診で、本人が後述している「`default: true` を再定義テーマ内に書くと問題がある」が本質。

---

## ✅ 実装推奨（4件・うち #4610 は #4534 と同一パッチで解消）

### [#4534](https://github.com/saadeghi/daisyui/issues/4534) — .tooltip が祖先の scrollWidth を膨らませファントムスクロールバーを生む（+ [#4610](https://github.com/saadeghi/daisyui/issues/4610) を同時に解消）

> 🔴 **`contain: layout` は却下が正解と実測で確定。** 代わりに anchor positioning が全エンジンで利用可能になっている（**Firefox 147 / 2026-01 で出揃い済み**）。

- **分類**: `css-bug` / **判定**: `valid`（確信度 `high`・実測） / **深刻度**: `low〜medium` / **難易度**: `moderate`（約130行のリファクタ）
- **`contain: layout` を採らない理由（実測）**: overflow コンテナ内の tooltip で計測 — 現行は Chromium 320/256・Firefox 318/256（幻影あり）。`contain: layout` は Chromium 261/256（解消）だが **Firefox 318/256 で全く効かない**（Gecko は絶対配置子孫を scrollable overflow から除外しない）。さらに**スクロール到達領域が消えるため Chrome では tooltip が視覚的に切れる**。uh-zuh の 2026-07-04 の指摘（Firefox で悪化・Chrome で切れる）は**事実**だった。
- **修正方針**: `.tooltip` に `anchor-name`/`anchor-scope` を付与し、`tooltip.css:9-20` の内容を **`position: fixed` + anchor positioning** に変更、位置決めロジック（68-201行）を `position-area` ベースに書き換える。**`@supports (anchor-name: --x)` でゲート**し非対応環境は現行の absolute 実装を維持（プログレッシブエンハンスメント）。**リポジトリ内に前例あり**: `dropdown.css:4` が `position-area`、`dropdown.css:84` が `@supports not (position-area: bottom)` を既に採用。
- **実測**: `position: fixed` + anchor では Chromium 261/256・**Firefox 257/256** と両エンジンで幻影が解消し、かつ tooltip が overflow コンテナ外に完全描画される。`anchor-scope` による複数インスタンスの分離も確認済み。
- **対応状況（MDN BCD）**: `anchor-name`/`position-anchor` = Chrome 125 / Firefox 147 / Safari 26、`position-area` = Chrome 129 / FF 147 / Safari 26、`position-try-fallbacks` = Chrome 128 / FF 147 / Safari 26。
- **#4610 が同時に解消する理由**: `position-try-fallbacks: flip-inline` が画面端で自動反転するため、**`tooltip-end` を JS で後付けする運用自体が不要**になり、iOS Safari の再レイアウト未実施バグを踏む経路が消滅する。
- **重要な整理**: daisyUI 側のブロッカーとされる「Safari が `popover="hint"` 未対応」（saadeghi, #3346, 2026-06-26）は、**top layer が必要な #3346 にのみ妥当**。**#4534/#4610 の scrollWidth 問題には popover は不要**で、この混同のため実装可能な修正が止まっていた。
- **留保**: (1) 1行修正ではなく tooltip.css 位置決め部の書き直し、(2) **Safari 実機未検証**（本調査環境に WebKit なし）、(3) `position: fixed` は祖先の `transform`/`will-change` で包含ブロックを奪われるため daisyUI 自身の drawer サイドバー内（`drawer.css:36,44`）では従来挙動に戻る。

### [#3568](https://github.com/saadeghi/daisyui/issues/3568) — Firefox でページ遷移時にスクロールが先頭に戻らない

> 🔴 **前提が変わっていた。** 報告時(Firefox 135)には View Transitions が無かったが、**Firefox 144(2025-10)でサポート済み**＝当時と今で分岐が逆になっている。

- **分類**: `docs`（SvelteKit 実装） / **判定**: `valid`（確信度 `medium-high`） / **深刻度**: `low` / **難易度**: `easy`
- **根本原因（再特定）**: `packages/docs/src/routes/(routes)/+layout.svelte:59` が **マウント時に無条件・恒久的に `document.documentElement.style.scrollBehavior = "smooth"`** を設定している。SvelteKit 2.68 のクライアント実装は遷移時に `scrollTo(0, 0)`（behavior 未指定＝CSS の `scroll-behavior` に従う）を呼ぶため、**docs では全ページ遷移のスクロールリセットがアニメーションになる**。アニメーション中に新ページの DOM 差し替えでドキュメント高さが変わると Firefox がスムーズスクロールを中断する。sulimanbenhalim の観察「長いページ→短いページで起きる」と一致し、彼が試した `window.scrollTo(0,0)` が効かなかった理由（その呼び出しも smooth 化される）も説明できる。
- **修正方針**: `afterNavigate` で、popstate とアンカー遷移を除外しつつ `scroll-behavior` を一時的に `auto` にしてから `scrollTo(0,0)`（より根治的には :59 の smooth 指定を CSS 側へ移し遷移中だけ `auto` にする）。
- **留保**: **Firefox 実機での検証は未実施**（本調査環境は Chromium のみ、Chromium では現象が出ない）。原因特定は「コード上の確定事実＋症状パターンの一致」に基づく強い仮説であり、**PR 前に Firefox で1回踏んで確定させるべき**。

### [#4436](https://github.com/saadeghi/daisyui/issues/4436) — Safari でレスポンシブ drawer を閉じる際に tooltip がちらつく

> 🟡 **緩和策として実装可能。** ただし WebKit バグのマスキングであり、**Safari 実機未検証**。

- **分類**: `browser-specific` / **判定**: `partially-valid`（確信度 `medium`） / **深刻度**: `low` / **難易度**: `easy`
- **根本原因（機序を特定）**: `packages/daisyui/index.js:46-49` の variant 定義により、**チェックが外れた瞬間（＝折りたたみアニメーション開始の t=0）に `.tooltip` が全サイドバー項目で有効化**される。一方 `drawer.css:41-43` は `translate 0.3s` / `width 0.2s` でアニメーション中。Safari はポインタ移動なしのレイアウト変化で hover ヒットテストを再実行しないため、古い `:hover` が即座に `.tooltip:hover` を満たし、アニメーション時間だけ tooltip が表示される。
- **修正方針（CSSのみ・JS不要）**: `tooltip.css:59` の表示側ディレイ `0s` ハードコードを `var(--tt-delay-in, 0s)` に変数化（**既定値 0s なので既存挙動は不変**）し、`drawer.css` に `.drawer-side .tooltip { --tt-delay-in: 0.35s }` を追加（`translate` の 0.3s より長く）。サイドバー内の tooltip だけに hover-intent 遅延が入り、アニメーション中のちらつきが隠れる。
- **留保**: 根治ではなく WebKit バグのマスキング。サイドバーの正当な tooltip も 350ms 遅れる。**Safari 実機未検証**。状態: saadeghi の 2026-02-16「I will work on it」以降 5か月以上更新なし。

---

## 📝 ドキュメント修正（4件・即対応可）

### [#4658](https://github.com/saadeghi/daisyui/issues/4658) — Ember のインストール手順が古い

> 🟢 **最も着手しやすい。** 報告者の理由は事実誤認だが、**手順自体は別の理由で確かに誤り**。実ブループリントを取得して検証済み。

- **分類**: `docs` / **判定**: `valid`（確信度 `high`・実ファイル確認） / **深刻度**: `low` / **難易度**: `trivial`
- **報告者の主張は誤り**: 「今の Ember に `index.html` は無い」→ `@ember/app-blueprint@7.1.1` の tarball を展開して確認したところ、**`files/index.html` はプロジェクトルートに存在する**（Vite 構成なので当然）。報告者が提案する `app.ts` への `import './styles/app.css'` も不要。
- **ただし当該手順は実際に壊れている**（`packages/docs/src/routes/(routes)/docs/install/ember/+page.md:53-61`）:
  1. **パス不整合** — 直前のコードフェンスは `app/styles/app.css` に書かせているのに、import 文は `"./app/styles.css"`（`styles/` ディレクトリが抜けている）。存在しないファイルを指している。
  2. **そもそも手動 import が不要** — ブループリントの `index.html` は既に `<link rel="stylesheet" href="/@embroider/virtual/vendor.css">` と `<link rel="stylesheet" href="/@embroider/virtual/app.css">` を出力しており、`app/styles/app.css` はこの仮想 `app.css` にバンドルされる。`<script type="module">` から CSS を import する手順は Embroider の配線と二重になる。
- **修正方針**: `+page.md:53-61` の「Import the CSS file in your index.html」ブロックを**丸ごと削除**する。手順は「Vite config に tailwindcss() を追加」→「`app/styles/app.css` に `@import`/`@plugin` を書く」の2つで完結する。
- **留保**: ブループリントのファイル構成と `index.html` の中身までは確認済みだが、**実際に scaffold してビルドが通るところまでは未検証**。PR 前に一度 `npx ember-cli@latest init --blueprint @ember/app-blueprint` を回して確認すべき。
- 状態: 2026-08-04 起票、メンテナ未回答。

### [#4648](https://github.com/saadeghi/daisyui/issues/4648) — Stimulus 向けインストールガイドの追加

> 🟢 **メンテナが明確にGOを出している。** saadeghi（2026-07-31）:「Let's add a new page. Don't edit the existing Rails page.」

- **分類**: `docs`（新規ページ） / **判定**: `valid`（確信度 `high`・メンテナ承認済み） / **深刻度**: `low` / **難易度**: `easy`（分量は多め）
- **スコープ（起票者が提示し、saadeghi が承認した内容）**: 新規1ファイル `packages/docs/src/routes/(routes)/docs/install/stimulus/+page.md`。Stimulus は CSS に干渉しないためセットアップ節は短く、Rails / Vite ガイドへのリンクが中心。以降が Stimulus 固有:
  - Theme Controller の localStorage 永続化（コンポーネントdocsが React 版へリンクしている箇所の Stimulus 版）
  - modal / dropdown / toast を小さな controller として実装する例（inline handler ではなく controller に寄せる）
  - Turbo Drive と morphing の注意点（`<dialog>` の open 状態、`<details open>`、初回描画前のテーマ復元）
- **注意点**: ロゴは saadeghi が追加すると明言済み（こちらで用意する必要はない）。**既存の Rails ページは編集しないこと**（明示的な指示）。新規ページ追加時は言語JSONの整合（`bun run lang:prune:write` 等）が必要かどうか、既存の install ページ追加コミットを確認すること。
- 状態: 2026-07-31 起票、同日 saadeghi が方針決定。未着手。

### [#4616](https://github.com/saadeghi/daisyui/issues/4616) — join の `--join-xx` 変数がドキュメントと実態で乖離

> 🟡 **docs 修正は今日可能／API 追加提案は 🎨 設計判断のまま。**

- **実測で「docs が誤り」を確定**: `.join` に `--join-ss/se/es/ee: 30px` を設定した場合の各子の computed `border-radius` — 最初の子 `4px|0|4px|0`（**無視**）／中間の子 `30px` 全周（**効く**）／最後の子 `0|4px|0|4px`（**無視**）。`join.css:21-40` の `:where(:scope > :first-child/:last-child/:only-child)` が子要素自身に宣言するため親からの継承を必ず上書きし、**中間の子だけ角丸が残って見た目が壊れる**。
- **実装可能な部分**: `packages/docs/src/routes/(routes)/docs/utilities/+page.md:277-280` の4行（`--join-ss/se/es/ee` を user-facing 変数として列挙）を削除、または「`.join` ではなく個々の `.join-item` に指定」と明記する。`--join-h`/`--join-v`（:275-276）は `.join` 指定で正しく効くので残す。※この表は翻訳JSONに全言語ぶら下がるため `bun run lang:prune:write` が必要。
- **設計判断のまま**: pdanpdan 提案の公開変数レイヤ（`--join-r`/`--join-r-ss` 等）の新設。新しい公開 CSS 変数 API の追加はメンテナ承認必須で、**saadeghi は本issueに一度も反応していない**（2026-07-06 起票以降）。

### [#4138](https://github.com/saadeghi/daisyui/issues/4138) — モバイルのキーボード表示で dock が押し上げられる

> 🟡 **本体（CSS）は daisyUI が直すべき問題ではない／docs 1行のみ実装可能。**

- **本体が対象外である理由**: 報告環境（Mi Browser・WeChat・QQ の古い Android Chromium WebView）は `interactive-widget` キーを解釈せず、報告者自身が両方の値を試して不可と報告済み（2025-09-29）。VirtualKeyboard API は Chromium 限定＋**JS 必須**で、CSS-only ライブラリである daisyUI 本体には入れられない。`dvh` は layout viewport のリサイズ自体を止められない。「画面高で dock を隠す」案は全ユーザーに影響する挙動変更＝設計判断。
- **実装可能な部分**: `packages/docs/src/routes/(routes)/components/dock/+page.md:38-40` に既にある viewport meta の INFO ブロックへ、`interactive-widget=overlays-content`（または `resizes-content`）を1行追記する（Chrome 108+/Firefox 132+ で有効、古い in-app WebView は無視する旨も併記）。**リポジトリ全体で `interactive-widget`/`keyboard-inset`/`virtualKeyboard` は1件もヒットせず完全に未ドキュメント**。issue をクローズする根拠にもなる。
- 状態: 2025-11-05 以降 8か月以上更新なし。`dock.css` の当該箇所に変更なし。

---

## 🎨 要設計判断（真に保留・3件）

### [#3804](https://github.com/saadeghi/daisyui/issues/3804) — select + checkbox のパフォーマンス

> 🔵 **論点が旧トリアージと違っていた。** 実測で `appearance: base-select` は**無罪**、真因は `:has()` ルール群。

- **分類**: `performance` / **判定**: `valid`（確信度 `high`・実測） / **深刻度**: `medium`
- **新規実測（Chromium 151・CDP Tracing、`.btn`×3000＋checkbox を20回トグル、`UpdateLayoutTree` 合計）**:

  | 条件 | 20回合計 | 1回あたり |
  |---|---|---|
  | A: daisyUI CSS そのまま | 643.8ms | 約 30–32ms |
  | B: `checked` を含む `:has()` ルール270本を削除 | 142.4ms | 7.1ms（**−78%**） |
  | C: `:has()` ルール全803本を削除 | 5.4ms | 0.27ms |
  | D: daisyUI CSS を丸ごと無効化 | 3.0ms | 0.15ms |

- **否定された仮説**: `:root:has(` を含む20本だけ削除しても**改善ゼロ**（saadeghi の 2025-04-21 の仮説は実測で否定、michaelkhabarov の反論が正しい）。`<select class="select">` をページから削除しても `UpdateLayoutTree` はほぼ不変（**base-select はトリガーではない**）。そもそも起票（2025-04-21）は daisyUI が base-select を入れた 2025-08-31 より前で、**スレッドは別々の2つの問題を混同**している。
- **設計判断の対象**: recalc の主因である `:has()` ルール約800本（`.tab:is(label:has(:checked))`、`.filter:not(:has(:checked...))`、`.swap:has(>:checked)`、`.collapse:not(.collapse-close):has(...)` 等）は、**「JS なしで状態を扱う」という daisyUI の設計思想そのもの**。削減・スコープ限定は公開挙動の変更になる。
- **✅ 並行して即実装可能（paint 側）**: `--fx-noise` のインライン化。`base/svg.css:2` の `:root { --fx-noise: url(...) }` を参照する9箇所（button/badge/checkbox/alert/fileinput/menu×2/radio/toggle）でローカル宣言に切り替える。**saadeghi 自身が 2025-04-25 に宣言しながら15か月未実装**。API 非破壊。Paint は A で 33ms/回 → D で 7ms/回 と再描画コストは実在。
- **旧トリアージの提案は非推奨**: 「base-select をオプトインクラスの背後にゲートする」は実測で recalc に効かず、#4549 が M151 で解決した今は base-select を外す動機も薄い。

### [#4310](https://github.com/saadeghi/daisyui/issues/4310) — base-select の幅が選択 option で変わりレイアウトシフトする

> 🔵 **「ブラウザの修正を待つ」は戦略として成立しないことを確認。** 世界は動いていない。

- **分類**: `css-bug` / **判定**: `partially-valid`（確信度 `high`） / **深刻度**: `low`
- **実測（Chromium 147 / 148 / 151 で同一）**: 短い option 選択時 `.select` 幅 73.7px → 長い option 選択時 **320px**（`clamp` の 20rem 上限）。`field-sizing` の computed value は3バージョンとも `fixed` で base-select には効かない。**新 syntax（`<button><selectedcontent>`）に移行しても変動する**（73.7px → 439.3px）ため、#4549 で推奨された markup 移行は本件の解決策にならない。
- **CSS プリミティブが存在しない**: base-select では `<option>` が `::picker(select)`（popover / top layer）内にあり select の intrinsic size に寄与しないため、「最も幅広の option に合わせる」は**純CSSでは原理的に不可能**。`field-sizing: content` は逆に「選択中の option に合わせる」機能。upstream（chromium/open-ui）にも解決の動きなし。
- **判断の対象（パッチ自体は1行）**: `@supports (appearance: base-select)` ブロック内で `width: 100%` / `min-width: 100%` にする（saadeghi が 2025-11-27 に希望を表明した案）＝**base-select の既定サイジングを「親を埋める」に変えるか否か**の一点。
- **ついでに見つかった独立のバグ**: `select.css:8` の `width: clamp(3rem, 20rem, 100%)` は、shrink-to-fit な親の中で **select が 320px なのに親が 441px に膨らむ**破綻を起こす（`100%` が循環的に max-content から解決される）。#4310 とは独立に修正価値あり（例: `width: 100%; max-width: 20rem; min-width: 3rem`）。
- 状態: 2025-11-27 以降コメントなし。

### [#4465](https://github.com/saadeghi/daisyui/issues/4465) — Range スライダーが Chromium で重い

> 🔵 **据え置き。issue 本文の前提自体が古い。**

- **分類**: `performance` / **判定**: `partially-valid`（確信度 `medium`） / **深刻度**: `low`
- **issue の記述が古い**: 本文が引用する `100rem` の box-shadow は**既に存在しない**。`810f519e`（2025-12-10、#4334修正）で `100cqw` 化され、これは issue 提出（2026-03-14）**より前**。さらに v5.6（2026-06-26）で `--range-fill-x`/`--range-fill-y`/`--range-fill-spread` に変数化＋`range-vertical` 対応済み。
- **新しい証拠はゼロ**: issue は 2026-03-16 以降4か月コメントなし。pdanpdan は代替実装まで作ったうえで「CPU 使用率の差は特定できない。有意な性能差を示す方法がないなら、元の方が読みやすく保守しやすい」と結論。報告者からの追加プロファイリングもなし。本調査でも新たな定量データは取得できなかった（headless 環境で `requestAnimationFrame` が発火せずフレームタイム計測不可）。
- **実務的な推奨**: 「issue 本文の前提（100rem）が既に古い」旨をコメントして再現手順を求める、またはクローズする。
- **未検証のアイデア（非推奨）**: `.range` に `container-type: inline-size` を付ければ `100cqw` が range 自身の幅に解決され塗り面積が激減する可能性。ただし UA shadow 内の `::-webkit-slider-thumb` での解決は未検証で、`range-vertical` の `100cqh` を壊すリスクあり。

---

## ⛔ 既存オープンPRあり（引き取り不可・2件）

> **重要**: 「停滞しているから引き取れる」という前提は**誤り**だった。**2026-08-06 再確認: 両PRとも `OPEN` / `MERGEABLE`**（#4395 は 2026-06-06、#4514 は **2026-08-01** に更新あり＝作者 pdanpdan は現役）。止まっているのは**saadeghi のレビュー**であり、別PRを出すと重複になる。

| Issue | タイトル | 既存PR | 状態 |
|---|---|---|---|
| [#4393](https://github.com/saadeghi/daisyui/issues/4393) | Inconsistent "Form requirement validator" docs example | [#4395](https://github.com/saadeghi/daisyui/pull/4395) | `clean`・head `8fd7bc2c`・レビュー0件。差分はわずか +4行（HTMLコメントの補足）。対象ファイルの master 側最終変更は 2026-02-16 で**衝突なし**。なお関連 #3573 は 2026-07-08 にクローズ済みで、議論の受け皿が現在ない |
| [#4513](https://github.com/saadeghi/daisyui/issues/4513) | Inconsistent styling when using prefix config with theme plugin | [#4514](https://github.com/saadeghi/daisyui/pull/4514) | `clean`・head `fbf68df1`。**バグ本体は設計議論と独立に修正可能**: `themePlugin.js:15` が `prefix` を受け取らない一方 `pluginOptionsHandler.js:26-27` は `${prefix}theme-controller` を使う非対称が原因で、PR の「`daisyui/theme` に `prefix` オプションを明示追加」は自動共有の議論とは無関係。必要なのは saadeghi の「テーマに prefix 要る?」への回答だけ（pdanpdan の答え「テーマは `.theme-controller` セレクタを出力するので要る」が正しい）。※「両プラグインで options を自動共有」は同一パッケージ内シングルトンで技術的には可能だが、プラグイン記述順依存・HMR での状態持ち越し・monorepo 多重解決の3点があり 🎨 設計判断 |

---

## 🟣 上流で解決済み（対応不要・1件）

### [#4549](https://github.com/saadeghi/daisyui/issues/4549) — `<select>` が Chrome 149 でオーバーフローし CSS が効かない

- **Chromium 側で修正済み**: CL **7919309「Remove overflow:visible!important for customizable select」** が **2026-06-10 に main へマージ**（コミットメッセージに `Fixed: 521434907`）。M151 ブランチポイント（2026-06-29）より前に着地＝**Chrome 151 に載る**。
- **実機で確認（同一ページ・daisyUI 5.7.4）**:

  | Chromium | `overflow-x/y` |
  |---|---|
  | 147.0.7727.15 | `visible`（UA の `!important` が勝つ） |
  | 148.0.7778.96 | `visible` |
  | **151.0.7922.34** | **`hidden`** ✅ |

  `select.css:26-28` の既存の `white-space:nowrap; overflow:hidden; text-overflow:ellipsis` が **Chrome 151 で初めて効くようになる**。**daisyUI のコードは1行も変更不要**。
- **2026-08-06 に stable 相当で再測定し解消を確認**（Chrome for Testing 151.0.7922.34、daisyUI 5.7.16）。300px のコンテナに issue 本文と同じ長文 option を入れた `<select class="select">` で:

  | 構文 | select の幅 | コンテナの `scrollWidth` |
  |---|---|---|
  | 旧構文（`<option>` のみ） | 280px | 296px（<300＝**はみ出さない**） |
  | 新構文（`<button><selectedcontent>`） | 280px | 296px |

  **旧構文でも CSS が効くようになった**ため、pdanpdan が回避策として提示した新 syntax への移行も `appearance-none` も不要。
- **旧トリアージの「難易度 hard／新 syntax 移行の採用判断が必要」は不要になった**。M151 stable（2026-07-28）で暫定回避の窓は閉じており、**本 issue は「Chrome 151 で修正済み」とコメントしてクローズ推奨**。なお 2026-07-05 の再オープンは M151 stable より前の時点の報告。

---

## 補足: 管轄外の側面を併せ持つもの

上記で扱った issue のうち、根本原因の一部がブラウザ側にあるもの（daisyUI 側でできるのは解消・緩和まで）:

- **[#4610](https://github.com/saadeghi/daisyui/issues/4610)** — `tooltip-end` 後付け時に iOS Safari がレイアウトを再計算しないバグ自体は WebKit 側の問題。ただし **#4534 の anchor positioning 移行により `tooltip-end` を JS で付け外しする運用そのものが不要になり解消**するため、✅ 実装推奨として #4534 と同一パッチで対応するのが筋（スレッドは「JSで付け外し」前提の堂々巡りになっている）。
- **[#4436](https://github.com/saadeghi/daisyui/issues/4436)** — hover 再評価は WebKit 側の問題。daisyUI 側は表示遅延による緩和まで。
- **[#4138](https://github.com/saadeghi/daisyui/issues/4138)** — 本体はレガシー Android WebView のビューポート挙動で対象外。docs 追記のみ実施可（📝 参照）。
