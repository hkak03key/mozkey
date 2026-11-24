Delegation session: e882716dbdcc47b89ee9b06f60f4a601
Codex model: gpt-6.2-astra
Codex effort: high
添付の `conclusion.md` は、本文から末尾の3つの「レビュー反映ログ」まで読めました。

## critical

**該当なし。中間段階の Tab の割り当てと、最終形の AI 予測をどの状態で使うかについて、方針決定に進めます。** 前回までの重要な訂正は保たれています。特に、① Shift+Tab や Space は即時の rejected ではなく最終確定との比較を予約する、② 手動 Tab の直後の通常文字では、その時点で session が採用済みの表記を確定する、③ 入力読みを延長する最終形では確定結果・feedback・Mozc 履歴の読みを決める必要がある、の3点は妥当です。ここでの「採用済み」は session 内の状態で、画面への描画完了を待つ判定ではありません。`session/session.cc:5832-5856,6460-6468,6579-6629,8497-8513,8904-8935`。[対象起点の session.cc](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc)

方針上、**Composition の Tab だけを変える案と、Conversion の Tab まで変える案は区別できます。** 前者では、5種の組み込みキーマップで Space 後と予測候補選択中の Tab を従来どおり残せます。後者では両方に及びます。`PredictAndConvert` 自体を変更する場合は、さらに Down、CUSTOM の Precomposition、マクロも対象です。`data/keymap/ms-ime.tsv:60,92,125-128,158-160`、`session/keymap.cc:1099-1121`、`session/session.cc:2795-2798,3480-3498,11135-11158`。[対象起点のキーマップ](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/ms-ime.tsv)

## should-fix

| 箇所 | 指摘と根拠 |
|---|---|
| §0、§1-1〜1-3、§1-8b、§2　サジェストと Down | **「窓がある」「同じ候補を選べる」「予測に入れる」は別の条件です。** 通常の Composition で Suggestion キーマップを選ぶ条件は converter の `SUGGESTION` 状態です。遅延ライブ保留中だけ保存した窓を見ます。非ライブ HOLD は実 converter の先頭 ID を直接フォーカスし、複製した読みと候補列との照合はライブ側だけです。失敗時には窓があっても `Predict()` に進みます。§1-1 の「手段がなくなる」は「その Composition から一打で入る既定キーを失う」が正確です。ms-ime では手動 Force 後に内部 live が続けば、窓なしでも Down で予測に入れます。`session/session.cc:3197-3212,3252-3271,3480-3498,10073-10147,11093-11132`、`data/keymap/ms-ime.tsv:31,60,92,159` |
| §1-5、§1-7、§1-10　手動 Tab 後の Space・Backspace | **応答待ちに Space を押した場合が抜けています。** ライブ off、通常版 Force、組み込み割り当てなら、応答待ちの Space は要求を消して Mozc の `CandidateNext()` に進みます。zenz 採用後の最初の Space は元の Mozc 表記に戻し、その一打で次候補へは進みません。また「ライブ on なら Backspace は一字編集」にはライブ変換由来という条件が要ります。設定 on でも、通常の Space 変換から手動 Force した場合は `Conversion Backspace = Cancel` を継承します。`session/session.cc:3224-3249,3323-3329,3345-3362,3443-3461,3501-3522,7332-7338,7401-7405,8439-8479,11039-11043`、`data/keymap/ms-ime.tsv:64,127,161-163` |
| §1-8〜1-10、末尾の未確認一覧　通常文字での Undo とゼロクエリ | **通常文字で zenz を確定する経路の Undo は、未確認から外せます。** 採用済みなら `CommitZenzLiveCorrectionResult` → `CommitStringDirectly` で、この確定用の Undo 状態を積みません。応答待ちなら `CommitInternal(false)` が一度積みますが、通常の次文字を composer に挿入すると `ClearUndoContext()` が走ります。また、採用済みの直接確定では `zero_query_suggestion` が有効なら `Suggest()` を呼び、応答待ちの通常文字確定では呼びません。§1-8 の「影響しない」は**当該状態の Tab 割り当て**に限定できます。その呼び出しで候補窓が実際に見えるかは未検証です。`session/session.cc:8497-8514,8909-8935,9167-9169,9553-9562,9618-9620,9816-9839` |
| §1-10、§2、§4　先頭サジェストの確定と feedback | **Down でフォーカスする操作のほかに、Shift+Enter で先頭候補を直接確定する操作があります。** 非ライブの通常サジェストではこれが残ります。手動 Force 後の CONVERSION では、内部 live が true でも設定 off のため `CommitFirstSuggestion` のライブ専用条件を満たしません。§4 の予約理由表には、候補 ID の選択、候補のクリック確定、表示中の先頭候補の確定も追加できます。いずれも該当経路で zenz が表示中なら key 一致を要求しない比較を予約します。予測中の Ctrl+Delete は別で、裏の候補の削除・抑制に成功すると `ConvertCancel()` が比較予約を破棄します。`data/keymap/ms-ime.tsv:158-162`、`session/session.cc:4050-4061,4281-4295,6460-6468,9640-9654,9702-9708,9722-9735,10681-10689,11073-11084` |
| §4　学習される単位 | 正式版 S3 の「投影した文節を先に試す」は確認できました。**その学習件数が 0 の場合は、全文の読み・表記に加え、逆変換由来の文節についても学習を試します。** 通常の確定時と互換経路の両方で同じ順序です。続き補完が履歴へ与える影響を見積もるなら、この二つ目の呼び出しも必要です。また、比較値を `result.value()` から取る説明には例外があり、取消後の句読点単打確定では、末尾の句読点を除いた表記を先に保存します。`session/session.cc:5804-5820,6480-6502,6552-6570,8774-8788,9182-9192` |
| §0、§3、§7　Force の意味と適格性 | §7-2 の「ForceZenz は Convert（全文変換）」は開始状態を書き添えるべきです。通常版が `Convert()` を呼ぶのは Composition からで、既に Conversion／Prediction なら現在の converter の preedit を使います。WithoutFeedback 版は既変換でも複製上で再変換します。また forced が省くのは自動実行用の条件です。応答には採用判定が残り、たとえば採用後に Mozc と同じ表記に戻った結果や、`CANDIDATE_ONLY` の英語表記はそこで退けられます。PASSWORD について添付で保証できるのは **composer の field type が PASSWORD なら Force は変換要求を出さない**ところまでです。Mac が Secure Event Input 時に送るイベントの `Context` と composer の状態が常に同期するかは未検証です。`session/session.cc:7324-7399,7430-7479,8076-8094,8142-8151,8166-8198,8215-8226,11735-11740`、`mac/mozc_imk_input_controller.mm:1444-1458` |
| §3、§5、§6、未確認一覧　確認範囲 | 正式版 U の採用は妥当ですが、本文では過大・過小の両方が残ります。**未検証**は engine の固定 footer と各 renderer での非表示、設定 GUI の有効・無効、GUI 側のコマンド列制約、Linux 推論可否、縦書きの個別規則、各テストの存在・期待値です。例は本文引用の `engine/engine_output.cc:401-405`、`gui/config_dialog/config_dialog.cc:9015-9055`、`command_sequence_editor_dialog.cc:140-163`、`session/zenz_named_pipe_client.cc:592-598`、`session/vertical_writing_key_transform.cc:57,99,150,160`。一方、Mac では手動出力の `live_conversion` をコントローラが受け取り、文節の強調・下線属性を付けず、非空なら選択範囲を未指定にするところまで**確認できました**。IBus の caret、Windows の要求長受け渡し、GUI の定義済みインポート3種も確認できました。OS 上の最終描画や TSF の取得順は未検証です。`session/session.cc:7882-7890`、`mac/mozc_imk_input_controller.mm:740-744,879-899`、`unix/ibus/preedit_handler.cc:102-115`、`win32/tip/tip_zenz_context_request.cc:34-40`、`gui/config_dialog/keymap_editor.cc:78-88,329-342` |

## nits

- **行番号を直す箇所**。`live_keymap_active` の式全体は `session/session.cc:3229-3231`、既存 zenz 状態のクリア呼び出しは `:7402`、Force の表記取り出しは `:7395-7399` です。最小読み長を飛ばす箇所は投入時の `:7476-7479` と応答検証時の `:8081-8084` の2か所です。文脈要求の4条件は `session/zenz_context_request.cc:25-29`、候補 ID **選択**の照合本体は `session/session.cc:4012-4053` です。正式版 N1 で見送った行修正のうち、ここは直すと追跡できます。
- **期限の境界**。defer 有効時は「期限を越えた最初の poll」より「期限**以上**の最初の poll」が正確です。通常の 3000 ms 側は `TakeResult()` の**後**に、未取得の場合だけ経過時間・poll 回数を検査します。両方を結果の破棄期限とは読めないようにできます。`session/session.cc:276-288,1167-1176,7901-7949`
- **auto-block の件数**。50% ちょうどを含む訂正は正しいです。分子は通常の rejected で、手動 hard reject はこの件数に入りません。短い読みの除外は実行時の自動再利用に効きます。管理用 `ListEntries()` は `auto_blocked` と `reuse_excluded` を別に算出します。`session/zenz_feedback_store.cc:603-610,642-661,1374-1385,1553-1573`、`protocol/config.proto:462-485`
- **Mac の通常文脈と IBus の補足**。Mac の左文脈は最大20 UTF-16単位で、文書長が1000単位を超えた場合や選択範囲が取得できない場合にも取得を見送ります。IBus が `preedit.cursor()` を使う条件には `live_conversion_pending()` も含まれます。`mac/mozc_imk_input_controller.mm:101-106,1211-1231`、`unix/ibus/preedit_handler.cc:102-115`

## 反映ログの判定

| ログ | 確認結果 |
|---|---|
| round 1 | **C1〜C3、S1〜S9 の採用は妥当**。保護スパン、Prediction の優先、WithoutFeedback でも残る手動 block、空 CUSTOM の既定へのフォールバックと条件付き migration を確認できました。S4 の未確認事項は上記の範囲まで解消できます。S8 の世代数は正式版 S8 の訂正が妥当です。**U の一部採用では今回の検証基準を満たしません。** N1 は上記の行修正が残り、N2 の製品既定値・メニューは確認できました。`session/session.cc:795-847,914-1042,3221-3250,8231-8258`、`session/keymap.cc:180-184,199-229`、`config/config_handler.cc:86-305,348-369,390-398` |
| round 2・旧版向け | **C の保持、S1〜S5、N1 の採用は妥当**。ATOK の Conversion Down は先頭文節確定、Windows の passive SUGGESTION は条件を満たすと通常描画へ進む、Mac の専用文脈取得には gate がある、Qt の左右 Shift は別イベントに由来する、IBus の Shift は modifier mask に由来することを確認できました。frontend 全体に横取りがないという断定を外した判断も妥当です。`data/keymap/atok.tsv:64,118-127`、`renderer/win32/window_manager.cc:252-288`、`mac/mozc_imk_input_controller.mm:516-520,1398-1425,1444-1477`、`gui/config_dialog/keybinding_editor.cc:69-70,186-205,354-375,518-531`、`unix/ibus/key_translator.cc:348-362` |
| round 2 正式版 | **S1、S2、S4、S7、S8、N2〜N4 は確認できました。** とりわけ `SetConfig()` 自体は投入済み要求を取り消さず、マクロ・遅延ライブ・zenz には別々の世代があります。**S3、S5、S6 は採用妥当で、上記の履歴学習、HOLD の条件、Undo・候補操作の補足が残ります。U は本文への反映が不完全、N1 の一部採用は結論を変えないものの行修正が残る**という判定です。`session/session.cc:2404-2410,3771-3775,4374-4384,7045-7063,7641-7669,10696-10744` |

手で選んだ候補や文節境界を採用ポリシーが最終的に守るか、続きを含む応答が通るか、Space 変換の順位がどう変わるか、実画面でちらつくかは、本文どおり**未検証**です。添付で確認できるのは、入力読み・基準表記・保護スパン・文節列を採用判定へ渡すところまでです。実行テストは行っていません。`session/session.cc:607-629,7390-7412,7535-7540,8142-8151`。[対象起点の採用判定呼び出し](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L8142-L8151)

[exited with code 0]
