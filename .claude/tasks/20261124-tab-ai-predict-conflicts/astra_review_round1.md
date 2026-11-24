Delegation session: 7e5bac7626fa441388c01dfd917cacc8
Codex model: gpt-6.2-astra
Codex effort: high
**判定は要修正です。** 添付の MS-IME 配列では、Composition の Tab を変えると、入力中に Mozc 予測を開く操作が変わります。Conversion の Tab も変えると、通常変換・予測候補の巡回にまで影響が広がります。ライブ変換を off にしても手動 zenz は起動できます。この骨格は確認できました。判断を変えるのは、**手動 zenz の応答を待たずに次の文字を打った場合の確定内容**と、**操作を rejected として記録する条件**です。

以下のパスは `src/` 基準です。公開資料では上流 Mozc のプリセット構成と mozkey の起点コミットを照合しました。判定は添付ソースの静的レビューに基づきます。テスト・実機確認は行っていません。[上流 Mozc の keymap 説明](https://github.com/google/mozc/blob/master/src/data/keymap/README.md)・[mozkey 起点コミットの session.cc](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc)

## critical

1. **§1-7・§4の「Shift+Tab／Space／矢印で rejected として学習」は誤りです。** 表示中の zenz から Shift+Tab で前候補へ進むと、`explicit_conversion_after_zenz` を理由に**最終確定との比較を予約**します。Space による復帰も同様です。記録が許可され、最終確定値が観測され、比較が確定した時点で、値が同じなら accepted、異なれば rejected、確定値がなければ中立です。途中の文節確定だけでは比較は完了しません。また MS-IME の Down は `PredictAndConvert` なので、手動 zenz の表示後でも予測用の専用分岐に入ります。「矢印」を一括して `explicit_conversion_after_zenz` とすることもできません。長い予測を確定して誤った負例が生じる可能性はありますが、将来の AI コマンドに同じ副作用が生じるのは、この比較経路を再利用した場合です。根拠は `data/keymap/ms-ime.tsv:92,125,159`、`session/session.cc:3349,3480-3522,6460-6503,6579-6629,8439-8441,10145-10161,11093-11124`。

2. **§0・§4は、Tab の直後に入力を続けると何が確定するかを過小評価しています。** 手動要求を投入した後、通常の文字入力が来ると、① zenz 表示済みなら zenz、②表示を保留していればその時点で見えていた preedit、③応答待ち・失敗で zenz が表示されていなければ Mozc の変換を確定し、次の入力を始めます。したがって同じ「Tab → 次の文字」でも、応答の到着順により確定文字列が変わります。これはコード上の分岐です。逆に、非ライブ状態から始めて投入前の適格性判定で落ちた場合は、一時的な live 状態を解除します。forced が省くのは自動実行の設定・遅延・最小読み長で、日本語文字を含まない読み、長すぎる読みや値、機微な文字列などの判定は残ります。句読点単打確定と Shift 英数は別経路です。根拠は `session/session.cc:1253-1254,1315-1395,7401-7421,7430-7479,7563-7574,8904-8935,8958-8991,9215-9238`。

3. **§1-5・§7-2の最終形では、読みの検証に加えて「何を確定し、何を学習するか」が未解決です。** 通常版 ForceZenz を Prediction から呼ぶと、`key` は composer の入力読み、`mozc_value` は preedit 各文節の**表記**を連結した値です。候補の読みはここでは使いません。「候補の読みが必ず長い」「必ず不一致」は確認できず、続きを含む候補を選んだ場合のリスクです。さらに、現行の zenz 確定処理は元の `key` と採用表記をそのまま確定結果・accepted の組にし、設定次第で Mozc 履歴にも渡します。仮に長い続きを表示できても、この経路を流用すれば「短い入力読み → 続き込みの全文」を学習し得ます。厳密な逆変換チェックは製品既定で off です。中間段階に Mozc の `Predict()` を呼ぶ処理がないことは確認できますが、全応答で続きを排除することや、別 validator が必須であることは未検証です。根拠は `session/session.cc:1664-1679,5786-5820,7336-7399,8080-8093,8142-8151,8215-8227,8296-8303,8497-8513,9816-9835`、`config/config_handler.cc:365-369`。

## should-fix

| 対象 | 修正すべき結論と根拠 |
|---|---|
| §0・§1・§2　Tab と Down の適用範囲 | 添付された配列は **MS-IME のみ**です。この配列で、窓なし Composition の `PredictAndConvert` は Tab、Down は `MoveCursorToEnd`。Suggestion では Down が `PredictAndConvert`、Shift+Enter が先頭候補の直接確定。Conversion にも独立に `Down = PredictAndConvert` があります。また、通常の非 HOLD Suggestion では `Predict()` に進むので、「いま表示中の同じリストを選ぶ」ことまで保証できるのは、HOLD または検証に成功したライブ側スナップショットです。**既存コマンド自体を AI 化する場合は Down や CUSTOM マクロにも波及し、Tab の割り当てだけを変える場合とは影響が異なります。** `data/keymap/ms-ime.tsv:31,60,92,125-129,157-163`、`session/session.cc:3197-3218,3252-3271,10070-10163,11135-11162`。 |
| §1-4　候補・文節の保護 | 「ユーザー辞書由来・ユーザー優先だけ」は誤りです。preedit 中の ASCII 識別子、条件を満たす数値リテラル、位置と出現回数などを照合したシステム固有名詞も保護対象に加わります。採用判定用の文節列は、読みを整合でき、連結表記も一致すれば元の境界を渡し、それ以外では全文一組に縮退します。任意の手選択や手動の区切り直しが守られるかは、採用ポリシー本体が未添付のため未検証です。`session/session.cc:607-629,795-850,914-1042,7538-7540,8142-8151`。 |
| §1-5・§1-6　Prediction 上に zenz が表示された場合 | 通常版は既存の Prediction を再変換しません。converter が PREDICTION のままなら、次のキーも **Prediction が ZenzConversion より優先**されます。MS-IME の Ctrl+Delete はこの場合 `DeleteSelectedCandidate` になり、表示中 zenz の明示ブロックに届きません。Space は別途、物理 Space と `ConvertNext` の特例で復帰できます。また手動 off でも内部の `live_conversion_active_` は立つため、残した `PredictAndConvert` はその間、通常の「次候補」ではなくライブ用予測経路に入ります。`data/keymap/ms-ime.tsv:127,157-162`、`session/session.cc:3224-3249,3349,3452-3461,3480-3498,7336-7404`。 |
| §1・§4　Backspace、候補操作、Undo の見落とし | MS-IME では自動ライブ中の Backspace は一字編集、off で手動 zenz を出した後の Backspace は通常 Conversion の `Cancel` を継承します。表示中なら、読み一致を要求する比較を予約して未変換の読みに戻ります。また内部 live 中はピン留め・解除が拒否され、候補 ID による選択・確定には対応する passive snapshot が必要です。通常文字扱いの候補ショートカットも、手動確定判定が先に走ります。さらに通常確定は Undo 用状態を積みますが、表示中 zenz の直接確定には同等の保存がありません。学習のロールバックと、確定前の表示を戻す Mozc Undo は区別が必要です。`data/keymap/ms-ime.tsv:64,163`、`session/session.cc:3341-3362,3412-3440,3920-3947,4012-4025,4090-4099,4224-4229,8487-8514,8909-8925,9115-9120,9553-9589,11073-11090`。 |
| §4　`WithoutFeedback` と最小読み長 | **どちらも添付で解決できます。** `WithoutFeedback` は変換時の統計的 feedback 参照を迂回し、非 incognito では明示 hard block を尊重します。採用後の commit は通常版と共通で、recording と Mozc 履歴学習の独立した設定・プライバシー条件に従います。既定の最小長 7 は rejected の**記録には効かず**、7 文字未満の自動再利用・auto-block に効きます。従って「1 回で auto-block」は、再利用対象の読みで、同一 `key/context_class/value` 内の件数・割合条件を満たす場合です。Space 変換の実順位への波及は rewriter 本体が未添付で未検証です。`session/session.cc:1202-1245,5859-5867,5912-5921,6378-6387,7344-7354,8231-8258,8509-8513`、`session/zenz_feedback_store.cc:36-49,642-661,1374-1385,1427-1432,1742-1751`、`protocol/config.proto:462-485`。 |
| §3　defer の期限 | 手動要求にも期限判定が掛かるという発見は正しいです。ただし、**設定フィールド未保存時は 300 ms** です。proto の既定は 250 ms ですが、実行時 getter は `has_...()` が false なら C++ 定数 300 を使い、明示値は 50–3000 ms に収めます。投入後、期限に達した最初の poll で `TakeResult()` **より先に**打ち切るため、早く回収できた応答は採用可能で、回収前に期限を越えた応答は既に届いていても捨て得ます。通常の 3000 ms はセッションの poll 待ち上限で、推論要求に渡す timeout とは別です。`protocol/config.proto:397-410`、`session/session.cc:272-288,1167-1176,7843-7856,7901-7924,7933-7949`。 |
| §3　ライブ off の追加影響 | off で止まるのは新規の自動ライブ開始・予約に加え、複数文節の学習済み zenz を即時表示する経路と、最小長待ちの `pre_live_conversion_reading` 表示フラグです。通常の `Convert` と `ConvertWithoutHistory` に渡す郵便番号リテラル優先設定も変わります。初回 Space の候補窓は、設定が有効で通常変換に成功した場合です。英数入力で Space を挿入・確定する分岐もあります。また、既に動いているセッションの `SetConfig()` 自体は zenz 要求を取り消さないため、「off にした瞬間に投入済み要求もすべて止まる」とは言えません。`session/session.cc:1113-1147,3771-3775,4549-4555,5342-5348,7065-7106,7820-7846,10692-10757,10764-10781,11191-11195`。 |
| §4・§7-1　裏計算と学習 | deferred の Enter は実際に見えていた preedit を確定します。一方、auto-block された非表示候補には、読み一致を要求する `auto_block_shadow_compare` が既にあります。「見せなければ学習しない」を全経路の性質にはできません。また全物理キーで必ず無効化するのは中断中の**マクロの世代**です。zenz には別の世代・読み・変換値・状態の照合があり、再要求や編集で取り消されます。裏計算の結果がどの操作で失効するかは、この二つを分けて判断する必要があります。`session/session.cc:2404-2413,5832-5856,6949-6999,7035-7063,7614-7747,8271-8283`。 |
| §0・§6　CUSTOM と overlay | 非空 CUSTOM は保存された表を読みますが、**空の CUSTOM は組み込み既定へフォールバック**し、その後 overlay が適用されます。「2 行だけで決まる」は専用行・overlay のない MS-IME の無修飾 Tab に限った説明です。既存 CUSTOM に新既定を配るかは製品判断です。既存 migration も一律追記ではなく、Space の元の割り当てや重複、特殊化した Delete の有無などを検査しています。`session/keymap.cc:180-184,199-242,1044-1121`、`config/config_handler.cc:145-239,286-299`、`protocol/config.proto:358-362`。 |

**未添付に由来する「確認済み」は、以下を未検証に変更してください。** とくに §5 の「全フロントで横取りなし」「影響は Mac のテストと描画に限る」は、今回の資料からは判断できません。

| 未検証の主張 | 添付から確認できる境界 |
|---|---|
| ATOK・Kotoeri・ChromeOS・mobile の Tab／Shift+Tab が同一 | 5 種のファイルを選ぶコードはあります。行の内容を確認できた TSV は `data/keymap/ms-ime.tsv:60,125,128` のみです。`session/keymap.cc:62-70,245-265`。 |
| Win TSF／Mac の拡張文脈取得と通常文脈へのフォールバック、実際の文脈短縮 | `GetZenzContextRequest` は live off、zenz off、取得不可、PASSWORD のいずれでも要求長を設定する前に戻ります。session は保存済み client context の view を使います。OS が収集する実テキスト、通常文脈への切替、精度差は未検証です。`session/zenz_context_request.cc:19-46,49-63`、`session/session.cc:7481-7500`。 |
| Win／Mac／IBus の Tab 横取り・キー翻訳、Win の ruby・窓抑制、Mac の描画、IBus の caret、Linux での zenz 利用不能 | 補正待ち・停止時・採用時の `output.live_conversion` と、出力候補窓をクリアして有効な passive 窓だけを戻す処理は確認できます。Linux で確認できる制限は添付の **feedback store の永続化**です。これだけで推論クライアントの可用性は決まりません。`session/session.cc:7750-7805,7811-7817,7880-7891,8349-8388,10241-10275`、`session/zenz_feedback_store.cc:1058-1066`。 |
| 「Tabキーで選択」の生成・各 renderer での非表示、設定 GUI の disabled／値保持、Backtab の取り込み、GUI のマクロ検証、縦書きの具体的なキー対応、各テストの存在・内容 | 添付で確認できるのは、session の footer 更新がフォーカスなしの窓では戻ること、コマンド列の**読込時**制約、縦書き変換関数の呼び出しまでです。§1-2・§3・§5・§6・§7 に挙がる各実装・テストの行は未検証です。`session/session.cc:11198-11208,11708-11732`、`session/keymap.cc:379-395`。 |

## nits

- **誤った行参照を直せます。** §1-2・§7-2 の `converter->Convert` は `session/session.cc:7374-7375`。§1-5・§1-6 の `Output`、読み取得、表記抽出、旧状態クリアは順に `:7390`、`:7391`、`:7397-7398`、`:7402`。§1-4 の文節列構築は `:7556` ではなく `:7538-7540`。§0 の遅延 0 の式は `:7563-7565`、§4 の `RecordRejected` 実呼び出しは `:6625-6629` です。
- §3 の製品既定値の代入は `config/config_handler.cc:353,356,397`、auto-block の件数 1・割合 50 は `protocol/config.proto:479-485` です。キーエディタの「プリセット 3 種」は**定義済み配列のインポートメニュー**の説明で、ファイルからのインポートは別にあります。`gui/config_dialog/keymap_editor.cc:78-88,329-342,710-721`。

## 確認できた

- LiveConversion、ZenzConversion、Suggestion、Prediction のフォールバック順。ZeroQuerySuggestion は Precomposition に戻るため、MS-IME の Composition／Conversion の Tab 変更は未入力時の Tab に直接波及しません。`session/keymap.cc:1044-1121`、`session/session.cc:3127-3158`。
- 通常 Conversion と通常の Prediction 選択中では、従来の Tab は次候補、Shift+Tab は前候補です。`data/keymap/ms-ime.tsv:125-129,158`、`session/session.cc:11039-11060,11150-11153`。
- ForceZenz の単発は通常のライブ変換解除より先に処理されます。通常版は既存 Conversion を再変換せず、`WithoutFeedback` 版は複製した context で再変換し、成功時に引き継ぎます。PASSWORD でこのコマンドに到達した場合は消費して終了します。`session/session.cc:3332-3339,3672-3687,7324-7382`。
- 表示中に ForceZenz を再実行すると、旧要求と表示状態をクリアして再要求し、そのクリア自体は rejected を予約しません。ちらつきの有無は実機未検証です。`session/session.cc:7045-7063,7389-7412`。
- キーエディタの状態一覧には LiveConversion・ZenzConversion・Suggestion・Prediction があり、ZeroQuerySuggestion は含まれません。読込時には ForceZenz 2 種はコマンド列の末尾のみ、Revert／Block は単独のみという制約があります。`gui/config_dialog/keymap_editor.cc:84-88`、`session/keymap.cc:379-395`。

[exited with code 0]
