Delegation session: 7e5bac7626fa441388c01dfd917cacc8
Codex model: gpt-6.2-astra
Codex effort: high
**round 1 の反映判定は保留です。** 今回のメッセージで受け取れたのはファイルパスの一覧で、更新版 `conclusion.md` の本文と末尾の「レビュー反映ログ」は確認できませんでした。追加指定のソース11件は、前回の報告が起点とした公開コミット `0eaeef17e` で照合しました。以下はその版での静的レビューです。更新後の worktree との一致や、実機での表示までは確認していません。[mozkey 公開リポジトリ](https://github.com/koyasi777/mozkey)

**「確認済み・astra 未検証」の併記は、誰が何を確認したかを残す目的なら妥当です。** ただし、同じ一文に「コード上の分岐」と「画面に実際に出る」が含まれる場合は、主張を分ける必要があります。たとえば Windows では、今回の資料で候補窓を隠す条件まで確認できました。ruby 窓の最終表示は引き続き未検証です。[window_manager.cc:252–288](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/renderer/win32/window_manager.cc#L252-L288)

### critical

今回追加されたソースの範囲では、**新規の critical は指摘なし**です。前回の重要事項については、更新本文がないため解消済みとは判定できません。特に、次の3点は修正文で条件が保たれている必要があります。

- Shift+Tab などで zenz を離れると、まず最終確定値との比較を予約します。その操作だけで rejected は確定しません。最終値が同じなら accepted、異なれば rejected、確定値を得られなければ中立になる分岐があります。[session.cc:3503–3519](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L3503-L3519)、[session.cc:6459–6468](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L6459-L6468)、[session.cc:6583–6628](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L6583-L6628)
- 手動 zenz の次に通常文字を打った際、確定対象はタイミングで変わります。zenz が既に可視ならその結果、表示保留中なら専用の確定処理、要求が未完了または失敗して Mozc が見えている場合は Mozc の結果です。句読点単打確定などには別経路があります。[session.cc:8905–8929](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L8905-L8929)
- 最終形で読みの続きを補う場合、既存の確定処理は `zenz_live_key_` と `zenz_live_value_` の組を accepted として予約し、同じ組を確定して Mozc 履歴学習処理を呼びます。入力済みの短い読みと補完後の長い表記をそのまま渡す設計では、validation に加え、何を確定キー・学習キーとするかが判断事項です。[session.cc:8487–8514](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/session/session.cc#L8487-L8514)

### should-fix

以下は**旧版の記述に対する再照合結果**です。更新版に同じ記述が残っていれば修正が必要です。

| 対象 | 照合結果 |
|---|---|
| **§1-1、§1-5、§2 の「Down が残る」** | 5プリセットとも **Suggestion の Down** は `PredictAndConvert` です。ただし通常変換の Down は、ATOK では `CommitOnlyFirstSegment`、mobile と ChromeOS では `ConvertNext` です。また ATOK の **Prediction** には `Ctrl Delete` のほか、Down / Up / Ctrl+Down / Ctrl+Up が明示されています。ATOK で「サジェスト窓があるときの代替キー」と「Space 後に押すキー」を混同すると、Down で最初の文節が確定します。[atok.tsv:64](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/atok.tsv#L64)、[atok.tsv:118–127](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/atok.tsv#L118-L127)、[mobile.tsv:92](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/mobile.tsv#L92)、[chromeos.tsv:82](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/chromeos.tsv#L82) |
| **§3 の Windows 候補窓・ruby 窓** | `live_conversion=true` で通常窓と cascading 窓を隠す分岐には例外があります。描画可能な本文を持つ、未フォーカスの非空 `SUGGESTION` が付いていれば、その非表示分岐を通りません。また `RubyWindow::OnUpdate` の呼び出しだけでは「手動 Tab 後に必ず ruby 窓が出る」とは確定できません。`ruby_window.cc` は今回の指定にも含まれていません。[window_manager.cc:247–288](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/renderer/win32/window_manager.cc#L247-L288) |
| **§3 の Mac・Windows の前後文脈** | Mac では live off にすると zenz 専用の `TEST_SEND_KEY` を省きます。通常の文脈取得にも、変換中文字列が空、対象アプリが対応、Secure Event Input が無効、といった条件があります。したがって手動 Tab に通常文脈が必ず届くとは言えません。Windows の指定ファイルで確認できるのは要求長の受け渡し、fallback の条件、取得不能時に専用欄を空にする処理までです。実際の TSF 呼び出し順や取得文字列は未検証です。[Mac:516–520](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L516-L520)、[Mac:1444–1486](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L1444-L1486)、[tip_zenz_context_request.cc:34–70](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/win32/tip/tip_zenz_context_request.cc#L34-L70) |
| **§5 の「フロントは Tab を横取りしない」** | Windows の `VK_TAB → TAB` と、対象イベントをサーバーに送り `consumed` を反映する経路は確認できました。Mac のコントローラはキー変換を `keyCodeMap_` に委譲し、変換失敗時や Direct モードの非対象キーでは送信前に戻ります。「調べた変換表・コントローラ内に Tab 固有の横取りは見当たらない」までが妥当です。全 frontend の処理や、Mac のテストへの影響まで確認済みとする根拠にはなりません。[keyevent_handler.cc:71](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/win32/base/keyevent_handler.cc#L71)、[同:945–978](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/win32/base/keyevent_handler.cc#L945-L978)、[Mac:1398–1425](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L1398-L1425) |
| **§6 の Qt の Shift+Tab** | 旧版が挙げた `:566` は Linux の「ひらがな／カタカナ」専用分岐です。Tab / Backtab はどちらも `Tab` に写し、別の Shift 押下イベントで保持した修飾状態を先に連結します。Windows では `LeftShift Tab` / `RightShift Tab` になり得て、GUI で左右共通にも変更できます。「常に `Shift Tab` として取り込む」は過大です。[keybinding_editor.cc:69–70](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/gui/config_dialog/keybinding_editor.cc#L69-L70)、[同:354–375](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/gui/config_dialog/keybinding_editor.cc#L354-L375)、[同:518–535](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/gui/config_dialog/keybinding_editor.cc#L518-L535)、[同:810–818](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/gui/config_dialog/keybinding_editor.cc#L810-L818) |

### nits

- IBus の `IBUS_ISO_Left_Tab → TAB` は確認できました。厳密には、`SHIFT` は keyval の名前から自動付与されず、非印字キーで `IBUS_SHIFT_MASK` が立っているときに付きます。キー変換表と修飾処理の両方を引用すれば十分です。[key_translator.cc:113–128](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/unix/ibus/key_translator.cc#L113-L128)、[同:348–362](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/unix/ibus/key_translator.cc#L348-L362)

### 確認できた項目

- **5プリセットの Tab 本体の割り当ては指摘なし。** Composition / Conversion の Tab は `PredictAndConvert`、Conversion の Shift+Tab は `ConvertPrev` です。追加4表にも Suggestion / Prediction / LiveConversion / ZenzConversion の Tab 専用行はありません。[MS-IME:60–128](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/ms-ime.tsv#L60-L128)、[ATOK:43–90](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/atok.tsv#L43-L90)、[Kotoeri:52–119](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/kotoeri.tsv#L52-L119)、[mobile:60–128](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/mobile.tsv#L60-L128)、[ChromeOS:50–107](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/data/keymap/chromeos.tsv#L50-L107)
- **IBus の caret は指摘なし。** preedit があり、出力が live または live pending なら `preedit.cursor()` を採用します。通常変換では `highlighted_position` があればそちらを優先します。ここは手動・自動の起動方法に依存しません。[preedit_handler.cc:97–115](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/unix/ibus/preedit_handler.cc#L97-L115)
- **Mac の候補窓抑制はコントローラまで確認できました。** live off では抑制関数は `false` を返します。さらに、手動 zenz の出力に preedit と `live_conversion=true` があれば、候補窓が空でも renderer を表示対象とする条件を満たします。最終的な ruby の描画は未検証です。[Mac:195–225](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L195-L225)、[同:993–1026](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L993-L1026)、[同:1170–1197](https://github.com/koyasi777/mozkey/blob/0eaeef17e/src/mac/mozc_imk_input_controller.mm#L1170-L1197)

round 1 の各指摘を「反映済み・一部反映・未反映」と判定するために不足しているのは、**更新版 `conclusion.md` の本文と反映ログ**です。旧版からの差分でも照合できます。

[exited with code 0]
