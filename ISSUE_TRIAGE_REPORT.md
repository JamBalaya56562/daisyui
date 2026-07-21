# daisyUI オープンIssue 調査レポート

> 対象リポジトリ: [`saadeghi/daisyui`](https://github.com/saadeghi/daisyui) / 更新日: 2026-07-21  
> 対象: 現在オープンな **Issue 全19件**（`master` @ `3198ac6`、v5.7.0相当）

## サマリー

| 分類 | 件数 | 対応方針 |
|---|---:|---|
| ✅ **実装推奨**（原因・修正方針が明確） | 2 | #4630／#4631 |
| 📝 **ドキュメント修正** | 1 | #3568（要設計） |
| 🎨 **要設計判断（保留）** | 10 | メンテナの方針決定が必要 |
| ⛔ **既存オープンPRあり** | 2 | #4393／#4513 |
| 🚫 **再現不能/仕様/バグでない** | 4 | #3872／#4138／#4488／#4610 |
| **合計** | **19** | |

---

## ✅ 実装推奨（2件）

原因・修正方針とも明確で、現行ソース（`master`）に問題コードを確認済み。

### [#4631](https://github.com/saadeghi/daisyui/issues/4631) — v5.6以降 `-xs`/`-sm` サイズ修飾子が効かない（レグレッション）

> 🔴 **最優先候補**。深刻度 high・難易度 trivial。ディスカッション [#4620](https://github.com/saadeghi/daisyui/discussions/4620) と重複（pdanpdan が同ディスカッションで原因を特定済み）。

- **分類**: `css-bug`（カスケード順序のレグレッション） / **判定**: `valid`（確信度 `high`） / **深刻度**: `high` / **難易度**: `trivial`
- **根本原因**: `e37aa036 "feat: daisyUI 5.6"` でサイズ実装が「各サイズが `--size` を直接指定」→「基底が `--size: calc(var(--size-field) * var(--sl-size-mul))` を計算し、各サイズは乗数のみ指定」に変更された際、既定値 `--sl-size-mul: 10` を **`.select, .select-md` という新ブロック（`@layer daisyui.l1.l2`）** に置いた（`select.css:301-304`）。5.6 以前は既定値が基底 `.select`＝より深い `@layer daisyui.l1.l2.l3` にあり、「深い入れ子レイヤほど弱い」daisyUI のレイヤ設計により `.select-xs`(l2) が自動的に勝っていた。5.6 では `.select-xs`(:275) / `.select-sm`(:288) と `.select, .select-md`(:301) が**同一レイヤ・同一詳細度 (0,1,0)** となり、ソース順が後の既定値ブロックが勝つ。`-lg`/`-xl` は後方にあるため無傷＝「`-xs`/`-sm` だけ効かない」という報告と一致。`--font-size-min`・`--option-px` も同時に潰れるため高さだけでなくフォントサイズも md 相当になる。
- **影響範囲（同一パターンの4コンポーネント）**: `select.css:301`、`input.css:245`（`--in-size-mul`）、`textarea.css:184`、`rating.css:66`。`btn`/`tabs`/`checkbox`/`toggle`/`radio`/`badge`/`range` は既定値を基底ブロック側に持つため**影響なし**。CDN 配布物（daisyui@5.6.18）の実バイトオフセットでも 4 件すべてで既定値ブロックが `-xs`/`-sm` より後方にあることを確認済み。
- **修正方針**: 既定値（`--sl-size-mul: 10` 等）を `.select, .select-md` のグルーピングから解き、**基底 `.select` ブロック（`@layer daisyui.l1.l2.l3`）へ移す**。`:301` は `.select-md` 単独で残す（＝5.6 以前の挙動と等価で推奨）。代替として既定値ブロックを `.select-xs` の前へ移動する案もあるがレイヤ設計上は脆い。input/textarea/rating にも同一処置が必要。回帰防止に xs〜xl が相異なる `--size` を計算するテスト追加が望ましい。
- **根拠**: CDN 配布物のバイトオフセット検証＋`sl log` による 5.6 の変更追跡。参照PRなし。pdanpdan が `Ref discussions/4620` をコメント。なお saadeghi は #4620 で当初「It works」と反応しているが、配布物の実バイト順で反証可能。

### [#4630](https://github.com/saadeghi/daisyui/issues/4630) — [Next.js] `.cally ::part(day):hover:not(selected, today)` でCSSパース失敗

- **分類**: `css-bug`（build-tooling 併発 / Lightning CSS パーサ） / **判定**: `valid`（確信度 `high`） / **深刻度**: `high`（Next 16/Turbopack でビルドが通らない＝ブロッカー） / **難易度**: `easy`
- **根本原因**: `calendar.css:33-37` の `::part(day):hover { &:not(selected, today) { … } }` がネスト展開され `.cally ::part(day):hover:not(selected, today)` になる。CSS 仕様上 `::part()` の直後に置けるのは user-action 系擬似クラスと `:state()` 等に限られ **`:not()` は不可**。加えて `selected`/`today` はクラスでも part 名でもない**裸の型セレクタ**で、part 名の否定は CSS では表現できない。Lightning CSS は `::part()` 後の `:not()` 引数を custom state と解釈しようとして `Invalid state` を投げる。browserslist を変えても効かないのは、トランスフォームではなく**パース段階の失敗**だから（報告者の指摘は正しい）。
- **二重のバグ**: セレクタ全体が無効なため、このルールは**ブラウザ上でも元々一度も適用されていない**（hover スタイルが全ブラウザで死んでいる）。
- **経緯**: 元は素の `::part(day):hover` ＋ `::part(selected):hover` だった。#4085 対応で `7077d2ac` が `:not(::part(selected), ::part(button today))`（これも無効）に置換、続く `f115ca9a` が `:not(selected, today)` に「修正」しつつ `::part(selected):hover` ブロックを**削除**。2回とも part 名否定を CSS で書こうとして失敗している。
- **修正方針**: 報告者のパッチ（`:not()` 削除のみ）は最小だが `::part(day):hover`(0,1,0) が `::part(selected)`(0,0,0) 等に勝つため **#4085 が再発**する。推奨は #4085 以前の構造への回帰＝ `:33-37` を素の `::part(day):hover { background: var(--color-base-200); }` に戻し、`:38-46` の `::part(button day today)` / `::part(selected)` の**後ろ**に `::part(button day today):hover` と `::part(selected):hover` を明示的に再宣言してソース順で上書きさせる。cally は custom state を公開せず part 名のみのため `:not()` による否定は原理的に不可能である旨をコメントで残すのが望ましい。
- **根拠**: `calendar.css` と `sl log` による経緯確認。timeline に参照PR・メンテナコメントなし。

---

## 📝 ドキュメント修正（1件）

### [#3568](https://github.com/saadeghi/daisyui/issues/3568) — Firefox: scroll position not reset to top when navigating between docs pages

> ⚠️ **見送り（要設計）**：提案修正（`afterNavigate` + `window.scrollTo`）は別コントリビューターが試して「効かない」と報告済み。Firefox の scroll-restoration と `startViewTransition` の競合で、簡単な docs PR では解決しない。

- **判定**: `valid` (確信度 `medium`)
- **根本原因**: docs レイアウトの onNavigate ハンドラ（`packages/docs/src/routes/(routes)/+layout.svelte:29`）が、ページ遷移に非同期の `document.startViewTransition` パターンを使っている。Firefox が DOM コンテンツ置換の前にスクロール位置を復元/設定する挙動（コメントで参照されている swup #95 / SvelteKit のスクロール回帰の癖）と組み合わさると、長いページから短いページへ遷移した際にウィンドウが先頭までスクロールされない。SvelteKit の既定のスクロール処理が View Transition のラップによって実質的に迂回・競合させられ、Firefox はそれを補正しない。
- **修正方針**: `packages/docs/src/routes/(routes)/+layout.svelte` で、アンカー遷移以外の遷移時にスクロールを明示的にリセットする。`afterNavigate` の import とハンドラを追加するか、`onNavigate` 内でスクロールする。例: `import { onNavigate, afterNavigate } from "$app/navigation";` として `afterNavigate((nav) => { if (nav.type !== "popstate" && !nav.to?.url.hash) window.scrollTo(0, 0); })`。参照されている Firefox/swup の癖はコンテンツ入れ替え前にスクロールを設定するため、long→short ページのケースではリセットを `requestAnimationFrame` でラップする（または `navigation.complete` 後に startViewTransition コールバック内で実行する）必要がある可能性が高い。タイミングの確認に Firefox での検証が必要なため、確実な1行修正ではなく設計・検証を要する変更。

---

## 🎨 要設計判断（保留・10件）

妥当な指摘だが修正にトレードオフ（既存挙動・パフォーマンス・対応範囲）があり、メンテナの方針決定が望ましいもの。

### [#3804](https://github.com/saadeghi/daisyui/issues/3804) — Performance: appearance: base-select on .select triggers expensive whole-document style recalculation in Chrome
- **分類**: `performance` / **判定**: `valid` (確信度 `medium`) / **深刻度**: `medium`
- **根本原因**: 複合的な2つの原因がある。(1) `appearance: base-select` のオプトイン（`select.css:76-98`）が Chrome の実験的な customizable-select 機能を有効化する。影響を受ける Chrome/Edge バージョンでは、無関係な操作でエンジンが広範な document 全体の「Recalculate Styles」を行う（Chrome 側のコストだが daisyUI がそれをオプトインしている）。(2) 多数のコンポーネント（button, badge, checkbox, alert, menu 等）が参照するグローバル共有の `--fx-noise` カスタムプロパティ（`base/svg.css:2`）があり、これを無効化すると全消費者の再描画が強制される（saadeghi が確認済み）。`:has()` を使う select 固有のセレクタがさらにマッチコストを増やす。select の症状はブラウザバージョン依存（Chrome macOS 135 で再現、Chrome Linux 132 では非再現）で、部分的に upstream の Chrome の癖でもある。
- **論点/方針**: `packages/daisyui/src/components/select.css` と base に対する独立した2つの緩和策。(1) `appearance: base-select` を全 `.select`（および子孫 `& select`）に既定適用するのをやめ、オプトインクラスの背後にスコープする — 例えば 78-80 行と 94-98 行を `.select-base` / 明示的な修飾子でゲートし、既定の `.select` は `appearance-none` を保って Chrome の base-select 再計算パスを避ける。(2) `--fx-noise` の再描画問題（saadeghi の指摘）については、グローバルの `--fx-noise` 変数（`base/svg.css:2`）を参照する代わりに、ノイズの url 値をコンポーネントごとにインライン化し、1要素での切り替え・無効化が全消費者を無効化しないようにする。いずれも base-select パスの除去・ゲートが主対象の挙動・設計判断であり、Chrome バージョン横断でのメンテナ検証が必要。

### [#4310](https://github.com/saadeghi/daisyui/issues/4310) — base-select width varies with selected option, causing layout shift
- **分類**: `css-bug` / **判定**: `partially-valid` (確信度 `medium`) / **深刻度**: `low`
- **根本原因**: `appearance: base-select` では、ブラウザが現在選択中の option を inline-flex な `.select` ボックス内の通常のインフローコンテンツとしてレイアウトする。daisyUI は `width: clamp(3rem, 20rem, 100%)` を設定しているが、これは単一の安定した幅ではなく、base-select ではボックスの実効的なコンテンツ由来サイズが選択 option によって変わるため、option 変更時に幅がずれる。ネイティブの select は最も幅広の option の幅を事前計算するが、base-select はしないため、純CSSで base-select を最も幅広の option に合わせる方法はない。メンテナ（saadeghi）はコメントでこれを望ましい機能拡張だと確認しており、pdanpdan も base-select が設計上流動幅であること、ネイティブの安定幅の再現が容易でないことを確認している。
- **論点/方針**: base-select でネイティブの最大幅サイジングを模倣する、クリーンで完全に正しいCSS修正は存在しない。メンテナが提案した部分的緩和策（親の100%を埋める）は、base-select ブランチをコンテナに引き伸ばす方法 — 例えば `.select` の `@supports (appearance: base-select)` ブロック内で `width: 100%`（または `min-width: 100%`）を設定し、要素が option コンテンツを追従する代わりに親を埋めるようにする。ただしこれは全 base-select select の既定サイジングを変える（挙動・設計変更）ため、ドロップイン修正ではなくメンテナの判断が必要。pdanpdan が指摘したように、ユーザーは明示的な `width`/`w-full` を設定すれば既に回避できる。

### [#4350](https://github.com/saadeghi/daisyui/issues/4350) — table-pin-rows overlaps multiple header rows (all get top:0)
- **分類**: `css-bug` / **判定**: `valid` (確信度 `high`) / **深刻度**: `low`
- **根本原因**: sticky 配置が各 `thead tr` に固定の `top: 0` で（また各 `tfoot tr` に `bottom: 0` で）一律に適用されている。CSS は各行の高さを知らずに行ごとのオフセットを自動的にカスケードできないため、複数のヘッダ行が同じ sticky 位置に重なってオーバーラップする。実質的に1行だけが固定されて見える。
- **論点/方針**: クリーンな汎用CSS修正は存在しない。N個のヘッダ行を積み重ねるには、先行行の累積高さに等しい行ごとの `top` オフセットが必要だが、それはオーサリング時には不明。メンテナ判断が必要な選択肢: (1) 各 `tr` ではなく `thead`/`tfoot` 自体を sticky にする（`border-collapse`/`separate` でのクロスブラウザの癖があり、既存レイアウトを壊す可能性）、または (2) pdanpdan が issue コメントで提案したように、新しいオプトイン修飾子 `table-pin-head`/`table-pin-foot` を導入し、`table-pin-rows` は既存アプリを壊さないよう変更しない。pdanpdan によれば、現在の挙動を変えると依存しているアプリを壊す。

### [#4436](https://github.com/saadeghi/daisyui/issues/4436) — Tooltips flash during responsive drawer open→close transition on Safari
- **分類**: `browser-specific` / **判定**: `partially-valid` (確信度 `medium`) / **深刻度**: `low`
- **根本原因**: `is-drawer-close:tooltip` の有効化条件はチェックボックスの状態と共に即座に切り替わるが、drawer の折りたたみはアニメーションする。遷移中にボタンがまだカーソル下にある間、Safari は `:hover` をアクティブに保ち、（表示遅延0sの）tooltip がちらつく。daisyUI コアCSSの欠陥というよりブラウザの hover 再評価の差だが、state のみのバリアントで tooltip をオンにするドキュメント例のマークアップに影響される。
- **論点/方針**: ここではコアコンポーネントCSSの変更で明確に正しいものはない。ドキュメント例 `packages/docs/src/routes/(routes)/components/drawer/+page.md` で、遷移中の時間帯だけ tooltip を抑制するのが最善。実用的な選択肢: サイドバーの折りたたみ完了後にのみ tooltip が現れるよう表示遅延を加える、tooltip の可視性を state ではなくポインタ位置でゲートする、または transitionend 駆動のクラスで遷移完了後にのみ `is-drawer-close:tooltip` を付与する（JSが必要）。根本原因が Safari が遷移中に `:hover` を保持することなので、確実な修正には例への JS transitionend ゲート、または Safari の癖として受容するかのいずれかが必要。コアCSS変更ではなく例の記述・調整を推奨。

### [#4465](https://github.com/saadeghi/daisyui/issues/4465) — Range slider box-shadow track-fill may cause Chromium drag lag
- **分類**: `performance` / **判定**: `partially-valid` (確信度 `medium`) / **深刻度**: `low`
- **根本原因**: 塗りつぶしトラックは、thumb 疑似要素上のオフセット付きスプレッド `box-shadow` で描画され、スライダの `overflow: hidden` でクリップされた 100cqw 幅の着色領域を塗る。一部の Chromium バージョンでは、急速なドラッグ中にこれが再描画負荷になり得る。しかしメンテナ自身のベンチマークでは代替案との測定可能な性能差は見つからず、主張されている深刻なラグがこのCSSに起因すると確認されていない。また issue が引用するコード（100rem）は現在のツリーと一致しない。
- **論点/方針**: 信頼できるベンチマークなしに変更は非推奨。メンテナは大きな `box-shadow` を避ける代替実装（issue コメントの tailwind play リンク参照）を同一の見た目で試作したが、測定可能な性能改善を示せず、現行手法の方が保守しやすいと判断した。対応するには (a) `box-shadow` がボトルネックであることを示す再現可能なプロファイリング手法と、(b) 明快さを代替案とトレードする旨のメンテナ判断が必要。進める場合、修正は `packages/daisyui/src/components/range.css` の 58-65 行と 88-95 行を pdanpdan の試作の gradient/clip ベースの塗りに置き換え、`--range-fill` / `--range-dir` の挙動を維持する。

### [#4534](https://github.com/saadeghi/daisyui/issues/4534) — .tooltip pseudo contributes to ancestor scrollWidth, causing phantom scrollbars
- **分類**: `css-bug` / **判定**: `valid` (確信度 `high`) / **深刻度**: `low`
- **根本原因**: 絶対配置された tooltip の疑似要素・子は常にレイアウトツリー内に存在し、ホストのボーダーボックスの外側に配置される。ブラウザは不可視（opacity:0）であってもそれをスクロール祖先の scrollWidth/scrollHeight に数える。daisyUI は配置だけではこれを抑制できない。ホストに `contain: layout` を付けるとオーバーフロー・スクロールへの寄与がホストにスコープされ、描画をクリップせずにファントムスクロールバーを解消できる。
- **論点/方針**: 候補となる1行変更: `packages/daisyui/src/components/tooltip.css` の `.tooltip` ルール内に `contain: layout;` を追加する。これにより tooltip 疑似要素がホスト外に描画されつつ、祖先の scrollWidth/scrollHeight への寄与を止められる。ただしメンテナ（pdanpdan）はこれを既定にすべきか明示的に疑問視し、オンデマンド（`contain-layout`）での適用や、ドキュメント化を提案しているため、直接マージではなくメンテナ判断が必要。**関連**: #4610（tooltip-end のモバイル余白）も同じ scrollWidth 寄与が背景にあり、本件の `contain: layout` 採用で付随緩和されうる。

### [#4548](https://github.com/saadeghi/daisyui/issues/4548) — Drawer crashes Chrome tab when opened after copying page text (scroll-driven animation)
- **分類**: `browser-specific` / **判定**: `valid` (確信度 `medium`) / **深刻度**: `high`
- **根本原因**: Chromium レンダリングエンジンのバグ: スクロール駆動アニメーション（`:root:has(.drawer-toggle:checked)` 上の `animation-timeline: scroll()`）が、drawer ボタン・サイドの `translate` 変形 + `will-change-transform` と相互作用する。テキストを選択・コピーして drawer をトグルすると、Chrome のコンポジタがタブをクラッシュさせる。CSS 自体は仕様上は妥当で、原因は Chrome 側（chromium.org の issue 522247742 で追跡）。daisyUI はエンジンのバグを直せず、回避策のみ可能。
- **論点/方針**: 機能を保ったままのクリーンなCSS修正はない。回避策はメンテナの設計判断とブラウザ検証が必要: (a) drawer でスクロール駆動アニメーションを避ける（`drawer.css:71-72`）、(b) ボタンの `translate`/`will-change-transform`（`drawer.css:40-44`）をアニメーションするルートから切り離す、または (c) `scroll()` アニメーションを機能・UA 条件でゲートする。いずれも採用前に Chromium の再現に対する検証が必要。

### [#4549](https://github.com/saadeghi/daisyui/issues/4549) — <select> overflows on Chrome 149, does not respond to CSS rules（再オープン）
- **分類**: `browser-specific`（upstream Chrome 寄り） / **判定**: `valid` (確信度 `medium`) / **深刻度**: `medium` / **難易度**: `hard`
- **根本原因**: Chrome 149 が `appearance: base-select` の select 描画挙動を大きく変更し、既存CSSが効かず overflow する。旧 [PR #4550](https://github.com/saadeghi/daisyui/pull/4550)（v5.6, 2026-06-16マージ）で一旦クローズされたが**再オープン**＝修正が不十分。upstream Chromium で追跡中（`issues.chromium.org/521434907`、修正は Chrome 151 予定）。
- **論点/方針**: 当面の回避は `appearance-none`。恒久対応は新しい `selectedcontent` syntax への移行（pdanpdan/timotejroiko が Play で確認）だが、base-select の記述方式変更を伴い対応ブラウザ差もあるため、メンテナの採用判断が必要。Chrome 151 の修正を待つ選択肢もある。
- **根拠**: issue コメント（timotejroiko/pdanpdan）、PR #4550 の状態確認。

### [#4616](https://github.com/saadeghi/daisyui/issues/4616) — DX: join utility class has documented --join-xx variables which are not really useful
- **分類**: `docs`（+ css-api-design / DX） / **判定**: `valid` (確信度 `high`) / **深刻度**: `low` / **難易度**: `moderate`
- **根本原因**: `join.css:3-6` で `.join` が `--join-ss/se/es/ee` を一律 `0` 初期化し、`@scope` 内の `:first/last/only-child`（:21-40）が `calc(var(--radius-field) * var(--join-h|v))` で上書きするため、`.join` 上でこれらを設定しても無効。さらに #4615 由来の `.join-item > * { --join-xx: initial }`（:45-51）で子孫への伝播も遮断。実質ユーザーが意味を持って設定できないのに、docs（`docs/utilities/+page.md:277-280`）は user-facing カスタマイズ変数として列挙しており実態と乖離。
- **論点/方針**: pdanpdan 提案の公開変数レイヤ導入が妥当（`.join` に `--join-r`/`--join-r-ss` 等を新設し、内部計算を `--join-ss: var(--join-r-ss, var(--join-r, var(--radius-field)))` 形へ変更）。または docs 表 :277-280 を削除して内部変数扱いに降格するだけでも整合が取れる。API 追加か docs 降格かは saadeghi の設計判断。#4615（close #4597）マージ済の follow-up。
- **根拠**: `join.css`/`docs/utilities/+page.md` を確認。参照PRなし。

### [#4618](https://github.com/saadeghi/daisyui/issues/4618) — Possible conflict between join/join-item and tooltip (z-index/display priority)
- **分類**: `css-bug`（stacking-context 起因） / **判定**: `valid` (確信度 `high`) / **深刻度**: `medium` / **難易度**: `moderate〜hard`
- **根本原因**: `join.css:15-19` の `@media (hover: hover) { > :where(.btn:hover, :has(.btn:hover)) { @apply isolate } }` が hover 中の join-item に新しい stacking context を生成する。tooltip（`tooltip.css:17` `z-index: 2`）はこの局所 stacking context 内に閉じ込められ、DOM 後方の兄弟 join-item のペイント順に勝てず覆われる。`join` を外すと isolate が付かず問題が消える。この isolate は #4320（group 内ボタン hover 時の右ボーダー）修正のため追加されたもので、saadeghi 自身が原因行を特定済み。
- **論点/方針**: isolate 除去は #4320 の回帰（hover 時にボタン端のボーダーが失われる、pdanpdan が Play で確認）を招くため単純除去不可。候補: (a) isolate をやめ hover 時ボーダーを別手法（`z-1` 昇格＋描画順調整）で再実装、(b) isolate は維持しつつ tooltip を含む子には stacking context を作らせない条件付け。副作用検証が必要で設計判断待ち。利用者は tooltip 付き join-item に `z-1` 付与で回避可（pdanpdan 提案）。現在 pdanpdan が対応調査中。
- **根拠**: `join.css`/`tooltip.css` 確認、issue コメント（saadeghi 原因特定、pdanpdan 調査中）。参照PRなし。

---

## ⛔ 既存オープンPRあり（2件）

| Issue | タイトル | 既存PR | 状態 |
|---|---|---|---|
| [#4393](https://github.com/saadeghi/daisyui/issues/4393) | Inconsistent "Form requirement validator" docs example | [#4395](https://github.com/saadeghi/daisyui/pull/4395) (OPEN, mergeable) | `valid` — 2026-06-06以降アクティビティなし。停滞しているがマージ阻害要因はなく、メンテナのマージ待ちのみ |
| [#4513](https://github.com/saadeghi/daisyui/issues/4513) | Inconsistent styling when using prefix config with theme plugin | [#4514](https://github.com/saadeghi/daisyui/pull/4514) (OPEN, base:master) | `valid` — 2026-05-03のメンテナ(saadeghi)との設計論争（`daisyui`/`daisyui/theme` 間でのprefix/root共有方法が未決）で停滞。PRは不完全な修正のため、そのままマージ不可 |

---

## 🚫 再現不能 / 仕様 / バグでない（4件）

| Issue | タイトル | 判定 | 理由 |
|---|---|---|---|
| [#3872](https://github.com/saadeghi/daisyui/issues/3872) | Countdown digits misaligned at 85% zoom in Safari Sequoia | `cannot-verify` | Safari が（85%ズームで生じる）非整数ピクセル値を、`overflow-y-clip` されたビューポート高さ（1em）と数字トラックのオフセットとの間で一貫せず丸めるため。純CSSトリック（各数字を縦に積み `-N em` でスライド）が非整数ズーム時のサブピクセル丸め累積に脆く、WebKit でだけ顕在化する。メンテナ(saadeghi)も「Safariのバグ」と判断済み。 |
| [#4138](https://github.com/saadeghi/daisyui/issues/4138) | Dock pushed up by software keyboard on legacy Android browsers | `invalid` | daisyUI のCSS欠陥ではない。レガシーな Chromium ブラウザ（Android の Mi/WeChat/QQ）はソフトキーボード出現時にレイアウトビューポートをリサイズするが、これは daisyUI では制御できない。 |
| [#4488](https://github.com/saadeghi/daisyui/issues/4488) | Custom theme colors overridden by built-in theme when using Vite in production builds | `cannot-verify` | daisyUI が出力するソースCSSに欠陥はない — `themePlugin.js` はカスタムトークンを組み込みテーマの上に正しくマージしている。報告された症状はソースだけからは再現できなかった。 |
| [#4610](https://github.com/saadeghi/daisyui/issues/4610) | Setting `tooltip-end` after render leaves extra space on right (mobile only) | `cannot-verify` / `partially-valid` | daisyUI 固有の CSS バグではなく iOS Safari の再レイアウト不具合。tooltip の擬似要素（`tooltip.css:9-20,73-75`）が親の scrollWidth を膨らませる構造（#4534 と同根）が背景で、JS でレンダ後に `tooltip-end`（`:152` の `inset-inline`）を付与すると iOS Safari がスクロール領域を再計算しない。saadeghi/pdanpdan もほぼ Safari 側問題と結論（回避: `max-sm:tooltip-right` 等）。#4534 の `contain: layout` 採用で付随緩和されうる。 |
