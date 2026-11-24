Delegation session: 08e6b74ba84b404cbdd5cf460a775682
Codex model: gpt-6.2-astra
Codex effort: high
添付の `conclusion.md` は、本文と末尾のレビュー反映ログ（round 1・2）まで読めました。

## critical

**進め方と方針の決定を止める critical はありません。** 「Tab の割り当てだけを変える案」と「`PredictAndConvert` 自体を変える案」は比較できます。特に、前者では既存の Down や専用の Suggestion 行を温存でき、後者では通常予測とライブ中の予測の両方が設計対象になります。中間案の ForceZenz が Mozc の変換結果を補正することも確認できました。上流 Mozc も Tab による前方一致予測を既存の操作として説明しています。`session/keymap.cc:1044-1121`、`session/session.cc:3480-3498,7332-7381,11093-11163`。[Mozc のキーマップ仕様](https://github.com/google/mozc/blob/master/src/data/keymap/README.md) · [Mozkey の調査起点の session.cc](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc)

ただし、**「Tab の次の文字で何が確定するか」と「一度の拒否で何が自動ブロックされるか」は、現文書の表現のまま仕様に採用すると操作結果を誤ります。** また、最終形で補完部分に与える読みと学習単位は、引き続き製品側で決める必要があります。以下の行番号は `src/` 相対です。

## should-fix

| 対象・ログ判定 | 照合結果と修正点 |
|---|---|
| **1. 手動 Tab 後の確定**（§1-9、R1 C2、R2 C） | C2 の採用は妥当です。判定時点を「応答の到着順」から**次の通常文字を処理する時点で、session が応答を採用し、zenz を表示状態にしたか**に改めてください。ワーカーに応答が届いていても、poll 前なら Mozc が確定し得ます。また、挿入関数には確かに 3 分岐ありますが、ライブ off から普通に手動 Tab を押す経路では、Force が既存の deferred 表示を消します。deferred 表示を新しく保存するのは自動ライブ経路です。この操作で説明すべき通常の結果は「表示済みの zenz」「それ以外は Mozc」の二つです。defer 設定による**要求の期限**は手動でも別に効きます。`session/session.cc:4549-4555,4816-4819,4853-4864,7045-7049,7401-7420,7901-7930,8296-8303,8904-8935` |
| **2. 自動ブロックの境界**（§4、R1 S5） | 「accepted の実績がない組に限る」は誤りです。既定の最小件数は 1、割合は **50% 以上**です。同一の読み・文脈クラス・表記に **accepted 1 件、通常の rejected 0 件**がある場合、次の rejected 1 件で `1 / (1 + 1) = 50%` に達します。件数・割合は設定可能で、短い読みの除外、再利用設定、incognito も適用されます。最小読み長が rejected の記録そのものを止めない点と、WithoutFeedback が非 incognito で手動ブロックを尊重する点は確認できました。`session/zenz_feedback_store.cc:603-609,642-659,1374-1385,1742-1750`、`protocol/config.proto:475-485`、`session/session.cc:1232-1245,8231-8258` |
| **3. 比較の予約と実際の学習**（§1-7・§4、R1 C1/C3、R2 C） | Shift+Tab・Space で比較を予約する訂正は妥当です。§4 の「長い候補を確定すると当然 rejected」はなお過大です。比較予約が残り、記録が許可され、観測した確定**表記**が異なる場合に記録します。読みの長さ自体は比較していません。また、部分確定中の出力は無視し、PRECOMPOSITION に戻ったコマンドの `result.value()` を取得します。先に確定した文節をこの関数で連結していないため、分割確定でも常に元の全文と比較できるとの保証はありません。最終形の Mozc 履歴も、まず文節単位の学習を試し、学習件数がゼロなら元の読み・全文表記の組へ進みます。本文の「学習しうる」は妥当で、この条件を足すと設計判断に使えます。`session/session.cc:5786-5821,6395-6422,6460-6502,6579-6629,9750-9781,10153-10155,11109-11113` |
| **4. Down の表と「窓があるときだけ」**（§0・§1-2、R1 S1、R2 S1） | **ChromeOS と mobile の Composition Down は、どちらも `MoveCursorToEnd`** です。表の `—` は修正が必要です。5 種共通なのは Suggestion の Down が `PredictAndConvert` に割り当たることです。通常の Space 変換後、MS-IME とことえりの Down は同名コマンドでも `ConvertNext` に進み、ATOK は先頭文節を確定します。一方、内部ライブ中の MS-IME では窓がなくても Down から新たに予測できるので、「窓があるときだけ」は **5 種共通の代替操作**に限定してください。なおコマンド自体の変更は ATOK の Suggestion `Ctrl+Down`、ことえりの Suggestion `Ctrl+n` にも届きます。`data/keymap/chromeos.tsv:31,82,120`、`data/keymap/mobile.tsv:31,92,159`、`data/keymap/atok.tsv:19,64,118-127`、`data/keymap/kotoeri.tsv:80,140-142`、`data/keymap/ms-ime.tsv:31,92,159`、`session/session.cc:3480-3486,11119-11125,11150-11157` |
| **5. コマンド自体を AI 化する場合の入口**（§1-3・§1-8・§2） | `Session::PredictAndConvert()` だけを変更すると、内部ライブ中には従来の Mozc 予測が残ります。`SendKeyConversionState` が `PredictAndConvertFromLiveConversion()` を直接呼び、そこから converter の `Predict()` にも直接進むためです。これはライブ off でも手動 Force 後に起きます。また、Precomposition／ZeroQuerySuggestion の「影響しない」は組み込み Tab の直接割り当てについては正しいものの、CUSTOM の Precomposition には `PredictAndConvert` を割り当てられます。表示リストの優先も、HOLD や窓の存在に加え、ID・読み・リスト・フォーカスの検証成功が条件です。`session/session.cc:2795-2798,3127-3135,3252-3271,3480-3498,7401-7412,10073-10101,10123-10147,11093-11132`、`session/keymap.cc:1090-1105` |
| **6. 候補削除・ピン留め・ID 選択・Undo**（§1-5・§1-10、R1 S3/S4） | Prediction 上の zenz で MS-IME の `Ctrl+Delete` がブロック操作に届かない点は正しいです。さらに、**背後の Mozc 候補に `DeleteOrSuppressCandidate` を実行し、成功時は変換を取り消します**。ピン留め・解除は内部ライブ中に拒否されます。ID 選択は表示したサジェストの複製と ID の整合が必要で、表示リストにない ID を背後の候補に流用しません。Undo も、少なくとも単発 Enter で表示中の zenz を直接確定する経路は追跡でき、この確定用の `PushUndoContext()` を通りません。通常確定、アプリ側の Undo、学習取消しとは区別できます。`data/keymap/ms-ime.tsv:157-162`、`session/session.cc:3143-3155,3233-3249,3385-3402,3920-3945,4009-4057,4090-4099,4224-4229,8487-8514,9561,9816-9840,10681-10689` |
| **7. ライブ off の直接効果**（§3、R1 S7） | `Session::SetConfig()` 本体は **`session.cc:3771-3775`** です。Undo 履歴を消して設定を差し替え、投入済み zenz 要求の取消しは呼びません。本文が挙げた `:7820-7846` は callback と poll 側です。通常 Convert／WithoutHistory の郵便番号優先は設定値で変わります。一方、通常 Convert の **英数 Space 分岐**の条件は入力モード、物理 Space、末尾の空白・カーソル、`space_on_alphanumeric` で、ここに `use_live_conversion` の直接判定はありません。初回候補窓にはライブ off 時の設定分岐とライブ on 時の郵便番号特例があります。`session/session.cc:3771-3775,5342-5348,7614-7745,7819-7846,10696-10757,10763-10774` |
| **8. 裏計算で使う世代**（§7、R1 S8） | マクロと zenz を区別した採用は妥当です。ただし「世代は 2 種類」は範囲が狭く、既存の遅延ライブ変換にも独立の世代があります。物理キーによるマクロ中断、遅延 Mozc 変換の callback、zenz 要求の callback は、それぞれ別に失効条件を持ちます。単一の callback 欄と Force をコマンド列末尾に限る制約は確認できました。`session/session.cc:2404-2410,2635-2638,4374-4384,5332-5339,5388,5492,7523-7525,7601-7611,7641-7644`、`session/keymap.cc:379-395` |

**R1 U の「一部採用」は、今回のレビュー条件には不十分です。** 「著者が読んだ」は来歴として残せますが、未添付の実装に依存する結論は**未検証**です。具体的には、固定 footer の生成と各 renderer での非表示、設定 GUI の各コントロール、コマンド列の GUI 側検査、Linux での推論可否、縦書き変換の具体的な矢印規則、各テストの存在・期待値です。たとえば添付の session 側で確認できるのは `CreateZenzClient()` を使うことと、縦書き時に矢印変換関数を呼ぶことまでです。逆に、今回添付されたキーマップ、Mac コントローラ、Windows のキー処理、IBus、キーマップエディタまで一括して「未検証」にする必要もありません。`session/session.cc:7811-7816,11708-11732`、`session/keymap.cc:379-395`。文書が挙げる未検証の参照先は `engine/engine_output.cc:401-405`、`gui/config_dialog/config_dialog.cc:9015-9055`、`gui/config_dialog/command_sequence_editor_dialog.cc:140-163`、`session/zenz_named_pipe_client.cc:592-598`、`session/vertical_writing_key_transform.cc:57,99,150,160` などです。

## nits

- **R1 N1 は採用妥当で、参照の補正が少し残ります。** `live_keymap_active` の全条件は `session/session.cc:3229-3231`、要求の読みは `:7391`、表記の抽出呼び出しは `:7395-7399`、抽出の本体は `:1664-1679` です。採用用の文節が不整合時に全文一組へ縮退する根拠は `:607-629` で、`:7538-7540,8142-8151` は組み立てた値の受け渡しです。最小読み長は要求前の `:7476-7479` と応答検証時の `:8081-8084` の両方で forced を扱います。
- **R2 S2 は採用妥当です。** Windows の有効な passive SUGGESTION は「その場で候補窓を隠して return する分岐」を回避します。その後の通常描画で窓を隠す経路があるため、§3 の「例外で、隠さない」は「通常の候補窓描画へ進む」が正確です。ruby の実表示を未検証とした判断は妥当です。`renderer/win32/window_manager.cc:252-288,393-401,665-667`
- **R2 S3 は採用妥当で、Mac の文脈はもう一段限定できます。** ライブ off で専用 preflight は省略されます。通常取得は文書長などの条件を通った場合の**左側最大 20 UTF-16 コード単位**で、この関数は右文脈を設定しません。Windows の添付ヘルパーでは要求長の保存・上限・fallback 条件まで確認でき、実際の TSF 呼出順と取得文字列は未検証です。`mac/mozc_imk_input_controller.mm:101-106,518-520,1211-1231,1444-1476`、`session/zenz_context_request.cc:19-35`、`win32/tip/tip_zenz_context_request.cc:20-25,34-53`
- **R2 S5 と N1 は確認できました。** Qt は Tab／Backtab を同じ `"Tab"` に写し、別に受けた Shift イベントから修飾名を組み立てます。末尾の「Shift+Tab がどう組み立てられるか」は解決済みにできます。IBus は `ISO_Left_Tab` を TAB に写し、実際の条件式 `IBUS_SHIFT_MASK && !IsPrintable(...)` で SHIFT を加えます。caret の分岐は `live_conversion_pending()` も対象です。`gui/config_dialog/keybinding_editor.cc:69-70,354-375,518-535`、`unix/ibus/key_translator.cc:66,113,127-128,348-362`、`unix/ibus/preedit_handler.cc:101-113`

## 確認できた項目

- **R1 C1・C3、R2 C の骨子**。比較を予約する訂正、予測の表記と入力読みを区別する訂正、続き込みの確定キー・feedback・Mozc 履歴を別々に決めるという論点は保持されています。R1 C2 の骨子も保持され、到達可能な分岐の訂正は上記 1 のとおりです。`session/session.cc:6460-6468,6579-6629,7390-7399,8497-8513,8909-8925`
- **R1 S2・S3・S6**。保護対象はユーザー辞書以外にも及びます。通常版を予測中に使った場合は Prediction keymap が優先されます。defer 期限はフィールド未保存時 300 ms、明示値は 50～3000 ms で、期限判定が `TakeResult()` より先です。`session/session.cc:795-850,914-1042,3224-3249,276-288,1167-1176,7901-7920`、`protocol/config.proto:402-407`
- **R1 S8・S9・N2**。auto-block で表示しなかった結果にも key 一致を要求する比較があります。空の CUSTOM は既定へ戻り、overlay は主キーマップの後に適用されます。既存 CUSTOM の移行は条件付きです。製品既定値の行、エディタの状態一覧、定義済み 3 方式とファイルインポートの区別も確認できました。`session/session.cc:8271-8283`、`session/keymap.cc:180-184,199-215`、`config/config_handler.cc:86-305,352-356,390-397`、`gui/config_dialog/keymap_editor.cc:78-88,710-721`
- **R2 S1・S4 の範囲限定**。組み込み 5 種の Tab／Shift+Tab と ATOK の特例は記載どおりです。Windows の `VK_TAB` 対応と `consumed` の反映、Mac のキー変換委譲・早期 return も確認できました。フロントエンド全体での横取りの不存在、ちらつき、実画面の結果は未検証のままです。`data/keymap/ms-ime.tsv:60,125,128`、`data/keymap/atok.tsv:43,64,88,90,118-127`、`data/keymap/kotoeri.tsv:52,116,119`、`data/keymap/chromeos.tsv:50,104,107`、`data/keymap/mobile.tsv:60,125,128`、`win32/base/keyevent_handler.cc:71,978,1073`、`mac/mozc_imk_input_controller.mm:1398-1425`

方針決定で最も効く境界は、**通常の Space 変換後や予測候補の選択中まで Tab を置き換えるか**です。Composition の Tab だけを変更すれば、組み込み設定では Suggestion の Tab も継承で変わり、Conversion／Prediction の Tab 巡回は残ります。両方の Tab 行を変更すると、予測候補を巡回していた Tab でも手動補正が始まります。この違いは添付コードで確定でき、実機描画や最終 validator の調査を待たずに決められます。`data/keymap/ms-ime.tsv:60,125,128,158-160`、`session/keymap.cc:1099-1105,1114-1121`、`session/session.cc:7332-7381,11135-11157`

[exited with code 0]
