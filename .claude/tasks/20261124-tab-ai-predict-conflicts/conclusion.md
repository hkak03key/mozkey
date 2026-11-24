# Tab を AI 予測変換 / ForceZenzLiveCorrection に置き換え、ライブ変換を切るときの競合調査

## 要約（ユーザー向け、15 行以内）

- **方針判断の分かれ目は、Tab をどこまで置き換えるか。** 案は 3 つあり、影響範囲が順に広がる。
- **案 1: Composition の Tab だけを変える**
  - 予測に一打で入る既定キーと、サジェスト窓を Tab で選ぶ操作がなくなる。窓が出ているときは Down で代わりになる（5 keymap 共通）。
  - Space 後と予測候補の選択中の Tab（次候補・巡回）はそのまま残る。
  - Force 後の 2 回目の Tab は、Conversion の Tab として Mozc 予測に入る（ライブ用の予測経路）。zenz が表示中なら、学習用の比較も予約する。
- **案 2: Conversion の Tab まで変える**
  - 案 1 の影響に加えて、Space 後の Tab が「次候補」から「現在の変換を元にした zenz 補正」に変わる。
  - 予測候補の選択中の Tab でも zenz が走る。入力済みの読みと予測候補の表記の組で zenz に渡る。zenz が表示された後も Prediction のキー設定が優先されるので、Ctrl+Delete で zenz をブロックできない。
- **案 3: PredictAndConvert というコマンド自体を変える**
  - さらに Down（全 keymap の Suggestion、ms-ime / kotoeri の Conversion）、ATOK の Ctrl+Down、kotoeri の Ctrl+n、CUSTOM keymap とマクロにも及ぶ。
  - 入口が 2 つある（通常の予測と、ライブ中の予測）ので、両方を作り直す必要がある。
- **どの案でも共通の注意**
  - Tab の直後の文字で何が確定するかは、その時点で zenz が採用・表示済みかで決まる。済んでいなければ Mozc の変換が確定する。
  - ライブ変換を切ると、Mac / Windows の zenz 用の前後文脈が取れなくなる。
  - defer 設定は、手動 zenz にも打ち切り期限として効く。
  - 予測で確定すると、zenz の補正は正しくても rejected と記録されうる。auto-block は 50% 以上で発動し、accepted が 1 件あっても、rejected 1 件で block されうる。
  - 最終形の続き補完では、確定結果の読み・feedback の読み・Mozc 履歴への反映を決める必要がある。
- **主な未確認**: 手で選んだ候補を採用ポリシーが守るか、続きを含む zenz 応答が validator を通るか、Space 変換の順位への影響、実機での描画とちらつき、Linux での推論。テストも実行もしていない。

- **調査対象**: worktree `20261124-background-live-conversion-tab`（origin/main 0eaeef17e 起点）。パスは `src/` からの相対パス。
- コードは変更していない（読み取りのみ）。
- **確度の表記**
  - 【確認済み】私がコードを読んで確認したもの
  - 【推測】コードから推したもので、実行やテストはしていない
  - 【astra 未検証】astra（確認役）に添付していない実装に依存するもの。私は読んでいるが、レビュー上は**未検証として扱う**。
    - 対象: engine の footer、renderer の非表示、設定 GUI の各コントロール、GUI のコマンド列検査、Linux での推論の可否、縦書き変換の規則、各テストの存在と期待値。
    - keymap、Mac のコントローラ、Windows のキー処理、IBus、キーマップエディタは、round 2 で astra も確認済み。
- round 1 のレビューを反映済み。反映ログは末尾にある。

---

## 0. 前提（競合を読むための土台）

### keymap の state 解決順【確認済み】

`session/keymap.cc:1044-1121`

- LiveConversion → Conversion
- ZenzConversion（ライブ中に zenz が表示されている）→ LiveConversion → Conversion
- ZenzConversion（それ以外）→ Conversion
- Suggestion → Composition
- Prediction → Conversion
- ZeroQuerySuggestion → Precomposition

### SendKeyConversionState が keymap を選ぶ優先順位【確認済み】

`session/session.cc:3220-3250`

1. `is_prediction`（converter が PREDICTION）なら Prediction keymap。**ZenzConversion より優先**される。
2. それ以外で `zenz_visible && live_keymap_active` なら ZenzLiveConversion keymap。
3. それ以外で `zenz_visible` なら ZenzConversion keymap。
4. それ以外で `live_keymap_active` なら LiveConversion keymap。
5. どれでもなければ Conversion keymap。

`live_keymap_active` は `live_conversion_active_ && live_conversion_from_composition_ && use_live_conversion()` で決まる（`:3227-3229`）。

### Tab の定義（5 keymap）【確認済み】

`ms-ime.tsv:60,125,128`、`atok.tsv:43,88,90`、`kotoeri.tsv:52,116,119`、`chromeos.tsv:50,104,107`、`mobile.tsv:60,125,128`

- 5 keymap とも次の 3 行で、ほかの state に Tab 専用の行はない。
  - `Composition Tab = PredictAndConvert`
  - `Conversion Tab = PredictAndConvert`
  - `Conversion Shift Tab = ConvertPrev`
- 適用範囲の注意
  - 「Tab の意味はこの 2 行で決まる」と言えるのは、組み込み keymap で、overlay も CUSTOM 専用行もない場合に限る。
  - **空の CUSTOM は組み込み既定へフォールバックする**（`keymap.cc:199-206`）。その後に overlay が適用される（`config.proto:358-362`）。

### Down の定義は keymap ごとに違う【確認済み】

| keymap | Composition の Down | Conversion の Down | Suggestion の Down |
|---|---|---|---|
| ms-ime | MoveCursorToEnd（:31） | PredictAndConvert（:92） | PredictAndConvert（:159） |
| kotoeri | （行なし） | PredictAndConvert（:80） | PredictAndConvert（:141）。Ctrl+n も同じ（:140） |
| chromeos / mobile | MoveCursorToEnd（:31） | ConvertNext（chromeos:82 / mobile:92） | PredictAndConvert |
| atok | Commit | CommitOnlyFirstSegment | PredictAndConvert |

atok は Prediction の Down も ConvertNext。

- 「Tab の代わりに Down で予測に入ればよい」と、**5 keymap に共通して言える**のは、サジェスト窓が出ているとき（Suggestion の Down）だけ。5 keymap とも、Suggestion の Down は PredictAndConvert になっている。
  - ms-ime では、内部ライブ中（手動 Force の後を含む）なら、窓がなくても Conversion の Down から予測に入れる（`:3480-3486`）。
  - Space で変換した後は、ms-ime と kotoeri の Down も、同じ PredictAndConvert でありながら実際には ConvertNext になる（`:11150-11152`）。
  - PredictAndConvert というコマンド自体を変える場合は、ATOK の Suggestion `Ctrl+Down` や kotoeri の Suggestion `Ctrl+n` にも影響が及ぶ。
  - Space で変換した後（Conversion）の Down は、keymap によって意味が違う。
  - ATOK では CommitOnlyFirstSegment（atok.tsv:64）なので、**Down を押すと最初の文節が確定する**。
  - mobile / chromeos では ConvertNext（次候補）。
  - ATOK の Prediction には、Ctrl Delete のほか、Down / Up / Ctrl+Down / Ctrl+Up の専用行がある（atok.tsv:118-123）。【確認済み】
- 「Tab の割り当てだけを変える」場合と「PredictAndConvert コマンド自体を AI 化する」場合とでは、影響範囲が違う。
- 後者だと、Down（ms-ime の Conversion / Suggestion など）や CUSTOM マクロにも波及する。

### ForceZenzLiveCorrection はライブ変換 off でも起動する【確認済み】

- forced は、自動実行のための次の 3 つを省く。
  - 設定 gate（`:7429-7435`）
  - 遅延（`:7563-7565`）
  - 最小読み長（`:8084`）
- 内部では `live_conversion_active_=true` を立てる（`:7400-7412`）。
- **残るのは適格性の判定**: 読みのプライバシー判定・値のプライバシー判定など（`:7437-7479`）。
- 落ちた場合、非ライブから始めていれば一時 live 状態を解除する（`:7413-7417`）。

---

## 1. Tab の既存の役割（state 別）と、置き換えたときにぶつかるもの

### 1-1. Composition（サジェスト窓なし）

- **現状**: Tab は Mozc 予測に入る入口（`session.cc:11135-11163`）。ms-ime では、窓がないときの Down は MoveCursorToEnd なので、**窓がない状態で予測に入れるキーは Tab だけ**。
- **競合**: Tab を ForceZenz にすると、その Composition から一打で予測（読みの続き）に入る既定キーがなくなる。
  - ms-ime では、手動 Force の後に内部 live が続いていれば、窓がなくても Conversion の Down で予測に入れる（§0）。
- 【確認済み】

### 1-2. Suggestion（サジェスト窓あり）

- **現状**: Suggestion keymap に Tab はないので Composition Tab にフォールバックし、PredictAndConvert になる。
  - 先頭で `FocusDisplayedPassiveSuggestion` が走る（`:11136`）。
  - 「表示中の同じリストを選ぶ」ことが保証されるのは、HOLD 中か、検証に通ったライブ側スナップショットのときだけ（`:10070-10163`）。
  - 通常の非 HOLD サジェストでは `Predict()` に進む。
- **競合**: ForceZenz にすると、窓が出ていても Tab で窓の候補を選べなくなる（ms-ime では Down で選べる）。
- 【確認済み】
- 「Tabキーで選択」の footer
  - engine が固定で出す（`engine/engine_output.cc:401-405`）。
  - Win / Mac / Qt の renderer はこれを非表示にする（`renderer/win32/candidate_window.cc:228`、`renderer/mac/CandidateView.mm:102`、`renderer/qt/qt_candidate_footer.cc:10`）。
  - 【確認済み・astra 未検証】

### 1-3. LiveConversion（ライブ変換 on）

- **現状**: Conversion Tab → `PredictAndConvertFromLiveConversion`（`:3481-3497`、`:11093-11132`）。
  - サジェスト窓があれば、そのスナップショットにフォーカスする。
  - なければライブ変換を破棄して Predict する。
  - 窓が見えているときは、Suggestion keymap の単発の PredictAndConvert / CommitFirstSuggestion が優先される（`:3252-3271`）。
- **競合**: ForceZenz にすると、ライブ中の Tab は「現在のライブ結果を baseline にして zenz を手動で再実行する」動作に変わる。
  - 単発の ForceZenz は、ライブ解除より先に処理される（`:3332-3339`）。
- 【確認済み】

### 1-4. Conversion（Space 後）

- **現状**: Tab = ConvertNext（`:11150-11152`）。
- **競合**: Tab が ForceZenz になると、現在の変換をそのまま baseline にして zenz を走らせる。通常版は Convert し直さない（`:7336-7381`）。
- 保護されるスパン（`BuildZenzProtectedConversionSpans`、`:914-1042`）
  - ユーザー辞書・ユーザー優先のほか、preedit 中の ASCII 識別子、条件を満たす数値リテラル、照合済みのシステム固有名詞も含む（`:607-629,795-850`）。
  - 採用判定用の文節列は、読みと連結表記が整合すれば元の境界を渡し、整合しなければ全文で 1 組に縮退する（`:7538-7540,8142-8151`）。
- ユーザーが手で選んだ候補や、手で区切り直した文節が守られるかは、採用ポリシー本体を読んでいないので【未確認】。

### 1-5. Prediction（予測候補を選択中）

- **現状**: Prediction keymap に Tab はない（ms-ime.tsv:158 は Ctrl Delete のみ）。Conversion Tab にフォールバックし、次の予測候補へ進む。
- **競合**: Tab = ForceZenz（通常版）にすると、予測中の preedit から zenz を走らせる。
  - key は composer の入力読み（`:7391`）。
  - mozc_value は preedit の各文節の**表記を連結したもの**（`ExtractPreeditKeyAndValue`、`:1664-1679`、`:7390-7393`）。候補の読みは使われない。
  - 選択中の候補が続きを含む予測（入力読みより長い）の場合、key と表記の対応がずれる。その場合に validation / adoption がどう振る舞うかは【未確認】。
  - 前回の「必ず不一致になる」は言い過ぎなので取り下げる。
- **予測の上に zenz が表示された後**【確認済み】
  - converter が PREDICTION のままなので、次のキーも Prediction keymap が ZenzConversion より優先される。
  - 例: ms-ime の Ctrl+Delete は `DeleteSelectedCandidate` になり、表示中の zenz を明示ブロックする操作（BlockVisibleZenzCorrection）に届かない（ms-ime.tsv:157-162）。
  - Space は、物理 Space かつ ConvertNext という特例で Mozc に戻せる（`:3452-3461`）。
  - 手動 zenz の間は内部の `live_conversion_active_` が立っている。そのため残った PredictAndConvert（ms-ime の Down など）は、通常の次候補ではなく、ライブ用の予測経路（`:3480-3498`）に入る。

### 1-6. ZenzConversion（zenz 表示中）

- 表示中に ForceZenz を再実行した場合【確認済み】
  - 旧要求と表示状態をクリアする（`:7397-7402`、`:7045-7063`）。
  - そのクリア自体は feedback の比較を予約しない。
  - そのうえで再要求する。
- Tab を連打して候補を巡回することはできない。
- Mozc→zenz の表示のちらつきは【推測】。

### 1-7. Shift+Tab

- `Conversion Shift Tab = ConvertPrev` のまま残るので、Tab と対称にならない。
- 表示中の zenz から Shift+Tab を押すと、generic promotion に入る（`:3503-3519`）。
  - **このとき予約されるのは、`explicit_conversion_after_zenz` を理由とした「最終確定との比較」**（`SetPendingZenzFeedbackComparison`、key の一致は要求しない、`:6460-6468`）。
  - 前回の「rejected になる」は誤りで、訂正する。
  - 確定のされ方で記録が分かれる（`:6579-6629`）。
    - 最終確定値が zenz と同じなら accepted。
    - 違えば rejected。
    - 全文の確定がなければ neutral。途中の文節確定では比較は完了しない（`:6490-6493`）。
- 【確認済み】

### 1-8. Precomposition / ZeroQuerySuggestion

- 組み込み keymap には Tab を直接割り当てた行がないので、影響しない。【確認済み】
- ただし CUSTOM の Precomposition には PredictAndConvert を割り当てられる（`:2795-2798`）。コマンド自体を変える場合は、ここにも影響が及ぶ。

### 1-8b. PredictAndConvert というコマンド自体を AI 化する場合の入口【確認済み】

- `Session::PredictAndConvert()` だけを変えても、内部ライブ中には従来の Mozc 予測が残る。
  - 内部ライブ中は、`SendKeyConversionState` が `PredictAndConvertFromLiveConversion()` を直接呼ぶ（`:3480-3498`）。
  - そこから converter の `Predict()` に直接進む（`:11093-11132`）。
  - これは、ライブ off で手動 Force をした後にも起きる。
- 入口は少なくとも 2 つあるので、両方を設計の対象にする必要がある。
- 表示中のリストが優先されるには、HOLD か窓が出ていることに加えて、ID・読み・リスト・フォーカスの検証に通ることが必要（`:10073-10147`）。

### 1-9. Tab の直後に文字を打ったときに何が確定するか（重要）【確認済み】

手動要求を投入すると `manual_zenz_commit_on_next_input_=true` になる（`:7418-7420`）。CONVERSION 中に次の通常文字を打つと、そのときの状態で確定内容が決まる（`:8904-8935`）。

1. zenz が表示済みなら、zenz を確定する（`CommitZenzLiveCorrectionResult`）。
2. 表示を保留中（deferred）なら、そのとき見えていた preedit を確定する。
3. 応答待ち、または失敗で zenz が表示されていなければ、Mozc の変換を確定する。

確定した後、その文字で新しい入力を始める。

- **ライブ off で普通に手動 Tab を押す経路では、2 の deferred は起きない。**
  - ForceZenz は `ClearZenzLiveCorrectionState()` を呼び、既存の deferred 表示を消す（`:7045-7049`、`:7397`）。
  - deferred を新しく作るのは、自動ライブ変換の経路だけ（`:4596-`）。
  - したがって、この操作で起こりうる結果は 2 つ。
    - 次の文字を処理する時点で、zenz が表示済みなら zenz を確定する。
    - そうでなければ Mozc を確定する。
- **「表示済み」になるのは、session が poll で応答を回収し、採用したとき。**
  - zenz のワーカーに応答が届いていても、poll より前なら Mozc が確定することがある。
  - 「応答の到着順」ではなく、「次の文字を処理する時点で採用・表示が済んでいたか」で決まる。
  - defer 設定による要求の期限（§3）は、これとは別に手動でも効く。【確認済み】
- 例外: 句読点の単打確定（`is_direct_commit_from_conversion`）と Shift 英数（`ShouldCommitLiveConversionBeforeShiftAsciiInput`）は別の経路を通る。
- 候補選択ショートカット（`MaybeSelectCandidate`、`:9115-9120`）よりも、この手動確定の判定が先に走る。

### 1-10. Backspace・候補操作・Undo

- **Backspace**【確認済み】
  - ライブ off で手動 zenz を出した後の Backspace は、通常の `Conversion Backspace = Cancel`（ms-ime.tsv:64）を継承する。
    - `live_keymap_active` が false なので、ライブ専用の一字削除経路（`:3349-3362`）には入らない。
  - zenz 表示中は、key の一致を要求する比較（`cancel_visible_zenz_to_composition`）を予約して、未変換の読みに戻る（`:3412-3440`）。
  - 一字編集になるのは、ライブ変換が由来の場合だけ（`live_conversion_from_composition_` が真で、設定も on）。
    - LiveConversion Backspace の行が効く。
    - 設定が on でも、通常の Space 変換から手動 Force した場合は、`Conversion Backspace = Cancel` を継承する（`:3227-3231`）。
- **Undo**
  - 通常の確定（`CommitInternal`）は `PushUndoContext()` を呼ぶ（`:9561`）。
  - zenz の直接確定（`CommitZenzLiveCorrectionResult` → `CommitStringDirectly`、`:8490-8514`、`:9816-9835`）では、関数内に PushUndoContext がない。
  - 単発の Enter で、表示中の zenz を確定する経路（SendKeyConversionState から直接呼ぶ、`:3385-3402`）でも、Undo 用の状態を積まない。【確認済み】
  - 確定前の表示に戻す Mozc の Undo と、学習のロールバックは別物として扱う必要がある。
- **ピン留め・解除**
  - 内部 live 中（`live_conversion_active_` / `pending`）は拒否され、DoNothing になる（`UpdatePinnedCandidate`、`:4090-4099`）。【確認済み】
- **候補 ID 選択**
  - 内部 live 中は、表示したサジェストの複製と ID が整合していることが必要。
  - 表示されていない ID を、裏の Mozc 候補に流用することはしない（`:4224-4229`）。【確認済み】
- **予測の上に zenz が表示されているときの Ctrl+Delete**（§1-5 の補足）
  - 裏の Mozc 候補に対して `DeleteOrSuppressCandidate` を実行する。
  - 成功すると ConvertCancel する（`:10681-10689`）。【確認済み】

---

## 2. サジェスト窓（passive suggestion）と Tab

- `FocusDisplayedPassiveSuggestion`（`:10070-10163`）は、次の 3 種を「表示されたまま」CONVERSION に昇格させる。【確認済み】
  1. 非ライブの HOLD 中のサジェスト
  2. ライブ中に cloned context で作ったサジェスト
  3. 遅延ライブの保留中サジェスト
- 呼び出し元は `PredictAndConvert` / `PredictAndConvertFromLiveConversion` だけ。ms-ime では、これが Tab と Down（Suggestion / Conversion）につながっている。
- **最終形で PredictAndConvert 自体を AI 化する場合**、この「表示中の窓を優先する」処理と AI 予測の、どちらを優先するかを決める必要がある。【未確定の設計論点】
- ライブ変換を off にすると、2 と 3 は生成されない（`MaybeStartLiveConversionInternal` は `:4549-4555` で即 false を返す）。【確認済み】

---

## 3. use_live_conversion=false で失われる・変わる機能

| 機能 | off での挙動 | 根拠 | 確度 |
|---|---|---|---|
| 自動ライブ変換と自動 zenz | 新しく始まらない。予約もされない | `:4549-4555`, `:7433-7435` | 確認済み |
| 複数文節の学習済み zenz の即時表示、最小読み長待ちの `pre_live_conversion_reading` フラグ | ライブ前提の経路なので働かなくなる | `:7065-7106`, `:11191-11195`（astra 指摘） | 一部確認（`MaybeApplyZenzFeedbackLiveCorrection` は冒頭を確認） |
| 通常の Convert / ConvertWithoutHistory の郵便番号リテラル優先 | 設定値で変わる。英数入力の Space 分岐には、use_live_conversion の直接の判定はない（入力モード・物理 Space・末尾の状態・`space_on_alphanumeric` で決まる）。前回、英数 Space 分岐も use_live_conversion で変わるように書いたのは誤りで、訂正する | `:10692-10757` は確認。`:10764-10781`, `:3771-3775`, `:5342-5348` は astra 指摘 | 一部確認 |
| ForceZenzLiveCorrection | 動く | §0 | 確認済み |
| 既に動いているセッションで設定を off にした場合 | `Session::SetConfig()` は、Undo 履歴を消して設定を差し替えるだけで、投入済みの zenz 要求は取り消さない。off にした瞬間に全要求が止まるわけではない | `:3771-3775` | 確認済み |
| zenz 用の拡張文脈の取得要求 | `GetZenzContextRequest` は、live off・zenz off・取得不可・PASSWORD のいずれでも要求長 0 で戻る。session は client context に保存された preceding / following を使う。**Mac** では、live off にすると zenz 専用の TEST_SEND_KEY 自体を省略する（`useZenzContextAcquisition_ = useLiveConversion_ && use_zenz_live_correction`、`mac/mozc_imk_input_controller.mm:518-520`）。通常の文脈取得にも条件がある（`:1444-1458`）。変換中の文字列が空（＝入力を始めるときだけ）であること、アプリが対応していること、Secure Event Input が無効であること。通常の取得で入るのは左側の preceding_text だけで、右側の文脈は設定されない（`:1211-1231`）。上限は 20 UTF-16 単位（astra が `:101-106` で確認。私は未読）。したがって、手動 Tab に通常の文脈が必ず届くとは言えない。**Windows** について確認できたのは、要求長の受け渡しまで。実際の TSF の呼び出し順と、取得される文字列は未検証 | `session/zenz_context_request.cc:14-25`, `session_handler.cc:455-484`, `zenz_client_context.cc:10-18` | gate は確認済み。OS 側の取得（`win32/tip/tip_zenz_context_request.cc:37-39`, `mac/mozc_imk_input_controller.mm:1469-1471`）は確認済み・astra 未検証。実際に文脈が短くなるか、精度が落ちるかは推測 |
| defer_live_conversion_display_until_zenz_result | ライブ経路では効かなくなる。ただし期限判定（`:7901-7924`）は forced かどうかを見ないので、手動 zenz も打ち切られうる。期限はフィールド未保存なら C++ 定数の **300ms**（proto の既定 250 ではない。`:276-278`, `:1167-1176`）。明示した値は 50〜3000ms に clamp される。期限**以上**になった最初の poll では、`TakeResult()` より先に打ち切る。そのため、応答が届いていても捨てられることがある。通常の 3000ms 側は、`TakeResult()` で結果が取れなかった場合にだけ、経過時間と poll 回数を見る。session の待ち上限で、推論要求の timeout とは別物 | `:7843-7856`, `:7901-7949` | 確認済み |
| GUI の zenz 関連コントロール | ライブ off で、自動補正・遅延・defer・最小長・ruby が disabled になる。値は保持される。右文脈と feedback 系は編集できる | `gui/config_dialog/config_dialog.cc:9015-9055` | 確認済み・astra 未検証 |
| show_live_conversion_ruby_window | 手動 zenz の出力は `live_conversion=true` を立てる（`:7764,7791,7882`）。そのため Windows の renderer は、ライブ off でも ruby window の更新経路に入る。通常の候補窓と cascading 窓は隠す分岐に入る。ただし、本文が描画できて、フォーカスがなく、空でない SUGGESTION が付いている場合は、その場で隠して return する分岐を通らず、通常の候補窓の描画処理に進む（`window_manager.cc:247-288`）。ruby window が画面に実際に出るかは未検証（`ruby_window.cc:105-108,293-296` は、設定値と `live_conversion` の gate まで読んだ） | 既定値は `config_handler.cc:353` | 分岐は確認済み（astra も確認）。実際の表示は未検証 |
| show_candidate_window_on_initial_conversion | 設定が有効で、通常の変換に成功したとき、初回の Space で候補窓が出る（`:10696-10757`）。ForceZenz は Session::Convert を通らないので、この設定は効かない | 既定値は `config_handler.cc:356` | 確認済み |
| Mac の候補窓抑制 | `useLiveConversion_` が false だと抑制しない。手動 zenz の描画は未確認 | `mac/mozc_imk_input_controller.mm:207-220,516` | 一部確認・astra 未検証 |
| ibus の caret | `live_conversion()` のとき preedit の cursor を使う | `unix/ibus/preedit_handler.cc:106` | 確認済み・astra 未検証 |
| Linux での zenz 推論 | `ZenzNamedPipeClient::IsAvailable()` は `_WIN32` 以外で false。Mac は Unix socket client（`zenz_client_factory.cc:102-108`）。Linux では Tab(ForceZenz) が zenz を実行できない。最終的な表示は推測 | `session/zenz_named_pipe_client.cc:592-598` | IsAvailable は確認済み・astra 未検証 |

---

## 4. zenz の feedback 記録・学習への影響

### 仕組み【確認済み】

- `SetPendingZenzFeedbackRejected(reason)` は、名前に反して即 rejected を記録しない。zenz 表示中に「**最終確定との比較**」を予約する（`:6460-6468`）。
- 確定時の判定（`:6579-6629`）
  - 記録が許可されていて、全文の最終確定値を観測できた場合:
    - 確定値が zenz と同じなら `RecordAccepted`。
    - 違えば `RecordRejected`（`:6625-6629`）。
  - 確定値がなければ neutral。
- `require_final_committed_key_match=true` の比較では、key が一致しないと neutral になる。

### 予約理由ごとの key 一致の扱い

| 予約理由 | key の一致 |
|---|---|
| Shift+Tab などの `explicit_conversion_after_zenz` | 要求しない |
| Space で戻す `space_revert_zenz_to_mozc`（`:8439-8441`） | 要求しない |
| `predict_after_zenz` / `predict_held_suggestion_after_zenz` | 要求しない |
| Backspace（Cancel）の `cancel_visible_zenz_to_composition` | **要求する** |

### 既存の誤学習の芽【確認済み】

- `predict_after_zenz` は key の一致を要求しない（`:11111`、`:10154`）。
- このため、zenz 表示中に Tab / Down で予測に入り、続きを含む候補を確定すると、rejected が記録されうる。補正そのものは正しかった場合でも同じ。
- 記録には条件がある。
  - 比較の予約が残っていること。
  - 記録が許可されていること。
  - PRECOMPOSITION に戻ったコマンドの `result.value()`（確定した**表記**）が、zenz の値と違うこと（`:6490-6502`）。
- 比べているのは表記で、読みの長さは比較していない。
- 文節を分けて確定した場合、先に確定した文節は連結されない。そのため、常に元の全文と比較できる保証はない。【確認済み】
- 最終形で Mozc 履歴に渡す場合の順序（`ConfirmPendingZenzFeedback` の互換経路など）。
  - まず文節単位で学習を試す（`MaybeLearnZenzProjectedSegmentsToMozcHistory`）。
  - 学習件数が 0 のときだけ、全文の読み・表記の組で学習する。【確認済み】
- 将来の AI コマンドでも同じ副作用が出るのは、この比較経路を使い回した場合。

### 最終形（AI 予測変換）で未解決の点：「何を確定し、何を学習するか」【確認済み（現行の経路）／設計は未確定】

- 現行の zenz の確定は、元の `key`（入力読み）と採用した表記を、そのまま確定結果にする。
  - accepted（`SetPendingZenzFeedbackAccepted`）とし、設定によっては Mozc の履歴学習（`ApplyPendingZenzAcceptedMozcHistoryLearning`）にも渡す（`:8497-8513`）。
- この経路を流用して続き込みの全文を確定すると、「短い入力読み → 続き込みの全文」という組を feedback と Mozc 履歴に学習しうる。
- 読みの逆変換を厳しくチェックする設定（`use_zenz_strict_reading_preservation`）は、製品既定で off（`config_handler.cc:365-369`）。
- 最終形では、少なくとも次の 3 つを別々に決める必要がある。
  - 続き部分の読み（確定結果の key）を何にするか
  - feedback の key を何にするか
  - Mozc 履歴に渡すかどうか
- 中間段階の ForceZenz には Mozc の `Predict()` を呼ぶ処理がない（`:7336-7399`）。【確認済み】
  - ただし、zenz の応答が続きを含まないことが全ケースで保証されているかは【未確認】。
  - 別の validator が必須かどうかも【未確認】。

### 自動 block と最小読み長【確認済み】

- `zenz_feedback_min_key_length`（Mozkey の既定は 7）は、rejected の**記録には効かない**。
  - 効くのは、自動再利用と auto-block の判定から除外するところ（`zenz_feedback_store.cc:1374-1385` の `feedback_key_length_excluded`）。
- auto-block の条件（`zenz_feedback_store.cc:642-661`、`config.proto:479-485`）
  - 有効で、同じ key / context_class / value の中で、
  - rejected の件数が 1 以上、
  - かつ rejected の割合が 50% 以上。
  - 既定は有効で、`config_handler.cc:397` で設定している。
- 前回の「accepted の実績がない組に限る」は誤りだった。訂正する。
  - 割合は **50% 以上**で判定する。
  - そのため、同じ読み・文脈クラス・表記の組に accepted 1 件・rejected 0 件がある状態でも、次の rejected 1 件で 1/(1+1)=50% に達し、block される。
  - 件数と割合は設定で変えられる。
  - block 判定には、短い読みの除外、再利用の設定、incognito も効く。【確認済み: `rejected*100 >= pct*observations`】
- Space 変換の順位にどう効くかは、rewriter 本体（`rewriter/zenz_feedback_candidate_rewriter.cc`）を読んでいないので【未確認】。

### WithoutFeedback

- 変換時に統計的 feedback を参照するのを迂回する（`:8231-8258`）。
- 非 incognito では、手動の hard block は尊重する。
- 採用後の確定は通常版と共通。recording と Mozc 履歴学習は、それぞれ独立の設定とプライバシー条件に従う。【astra の指摘を受け、`:8237-8247` を確認済み。確定後の共通経路は astra 指摘】

### （候補）裏で先に計算する場合

- 既存の扱い
  - deferred 中の Enter は、実際に見えていた preedit を確定する（`:3374-3388`）。
  - ただし、auto-block されて表示しなかった候補にも、key 一致を要求する影の比較（`auto_block_shadow_compare`、`:8271-8283`）は既にある。
- したがって「見せなければ学習しない」は全経路に共通の性質ではない。裏で計算した結果をどう学習するかは、経路ごとに決める必要がある。【確認済み（2 経路）・設計は未確定】

---

## 5. 各 OS frontend での Tab の特別扱い【確認済み・astra 未検証】

- **Windows**
  - `VK_TAB → KeyEvent::TAB` の変換だけ（`win32/base/keyevent_handler.cc:71`、`keyboard.cc:219,845,1058`）。
  - win32/tip に Tab 専用の分岐はない（grep した範囲で）。
- **Mac**
  - `kVK_Tab → TAB`（`data/preedit/mac-specialkeys.tsv:2`）。本体に Tab 専用の分岐はない。
  - テストに `TabShowsPredictionCandidateWindowDuringLiveConversion`（`mac/mozc_imk_input_controller_test.mm:731-`）がある。
  - zenz 文脈の preflight テスト（`:1391-1466`）は Tab を前提にしている。
- **ibus**
  - `IBUS_Tab` / `KP_Tab` / `ISO_Left_Tab` → TAB（`unix/ibus/key_translator.cc:66,113,127-128`）。
  - Shift は keyval の名前から付けるのではなく、非印字キーで `IBUS_SHIFT_MASK` が立っているときに付ける（同ファイル `:348-362`、astra が確認）。
- **結論（範囲を限定）**
  - **調べたキー変換表とコントローラの中には、Tab に固有の横取りは見当たらない。**
  - Windows は `VK_TAB → TAB` で送り、サーバーが返した `consumed` を反映する。
  - Mac はキーの変換を `keyCodeMap_` に任せていて、変換に失敗した場合などは送る前に戻る（`mac/mozc_imk_input_controller.mm:1398-1425`）。
  - ただし、frontend のすべての処理を網羅したわけではない。
  - Mac のテストへの影響は「Tab を前提にしたテストがある」ところまでしか確認していない。

### Tab のちらつきについて

- frontend 起因のちらつきは、範囲外として調べていない。

---

## 6. config dialog / keymap editor

- **キーマップエディタの状態一覧**（`gui/config_dialog/keymap_editor.cc:84-88`）【確認済み（astra も確認）】
  - DirectInput / Precomposition / Composition / Conversion / LiveConversion / ZenzConversion / Suggestion / Prediction。
  - ZeroQuerySuggestion は含まれない。
- **定義済みの keymap をインポートするメニュー**【確認済み・astra 未検証】
  - ATOK / MSIME / KOTOERI の 3 つだけ（`:78-82`）。
  - ファイルからのインポートは別にある（`:329-342,710-721`）。
- **Shift+Tab の取り込み**【確認済み】
  - Tab と Backtab はどちらも "Tab" に写す（`keybinding_editor.cc:69-70`）。
  - 修飾キーの名前は、別に届く Shift 押下イベントで保持した `shift_key_name_` を先頭につなぐ（`:354-375`）。
  - そのため、Windows では `LeftShift Tab` / `RightShift Tab` として取り込まれることがある。
  - 前回引用した `:566` は、ひらがな／カタカナ専用の分岐だった。訂正する。
- **ForceZenz 系のコマンド列の制約**【確認済み】
  - ForceZenz の 2 種は、コマンド列の末尾にしか置けない。Revert / Block は単独でしか使えない。
  - 読込時（`keymap.cc:379-395`）と GUI（`command_sequence_editor_dialog.cc:140-163`、astra 未検証）の両方で検査している。
- **CUSTOM keymap への配布**【確認済み】
  - 中身のある CUSTOM には、組み込み tsv の変更が届かない。
  - 既存の migration（`config/config_handler.cc:86-305`）は一律の追記ではない。元の割り当て、重複、特化した行の有無を検査している。
  - 既存の CUSTOM に Tab の新しい既定を配るかどうかは製品判断。配る場合は、既存の Tab 行の扱いを含めて migration を設計する必要がある。【未確定】

---

## 7. その他の競合

1. **callback と世代の管理**
   - `Output::Callback` は SessionCommand を 1 つしか持てない（`keymap.cc:379-382` のコメント）。
   - 世代は少なくとも 3 種類あり、それぞれ別の失効条件を持つ。
     - 物理キーで必ず無効になる、中断中の**マクロの世代**。
     - 遅延ライブ変換（Mozc）の callback の世代。`live_conversion_generation_` で、`CancelPendingLiveConversion` が進める（`:4374-4384`）。
     - zenz 要求の世代。世代・読み・変換値・状態で照合し、再要求や編集で取り消される（`:7614-7747`、`:7045-7063`）。
   - 裏で計算した結果がどの操作で失効するかは、これらを分けて設計する必要がある。【一部確認・設計は推測】
2. **中間案の ForceZenz は Predict を呼ばない。**【確認済み】
   - 通常版が `Convert()` を呼ぶのは、Composition から起動したときだけ（`:7374-7375`）。
   - Conversion / Prediction から起動した場合は、現在の converter の preedit をそのまま使う。
   - WithoutFeedback 版は、変換済みの状態からでも、複製した context 上で再変換する。
   - forced が省くのは自動実行のための条件だけで、応答の採用判定は残る。
     - 例: 採用後に Mozc と同じ表記に戻った結果や、`CANDIDATE_ONLY` の英語表記は、そこで退けられる（`:8142-8226`、astra 指摘）。
   - 最終形の続き補完では、読みの検証に加えて、§4 の「何を確定し、何を学習するか」が未解決。
3. **PASSWORD 欄**
   - ForceZenz は consumed にして何もしない（`:7329-7331`、`:3672-3687`）。【確認済み】
4. **縦書きのキー変換**
   - `session/vertical_writing_key_transform.cc:57,99,150,160` は、PREDICT_AND_CONVERT のバインドに依存している。
   - 依存しているのは Down 系で、Tab ではない。【確認済み・astra 未検証】
   - PredictAndConvert 自体を AI 化する場合は影響する。
5. **テスト**
   - `session/session_test.cc`、`keymap_test.cc`、`session_regression_test.cc` に、Tab / PredictAndConvert を前提にしたものがある。
   - 存在は確認したが、件数と中身は未確認。【astra 未検証】
6. **bypass 版の扱い**
   - bypass 版（WithoutFeedback）は、CONVERSION 中でも、複製した context で再変換して引き継ぐ（`:7336-7371`）。
   - 通常版は再変換しない。【確認済み】

---

## 8. round 3 の補足（Space・Undo・先頭サジェスト・学習単位など）【確認済み（特記がない限り）】

- **手動 Tab 後、応答待ちのうちに Space を押した場合**（ライブ off・通常版・組み込み keymap）
  - 要求を取り消して、通常の Mozc 次候補（ConvertNext）に進む。
  - zenz が採用された後の最初の Space は、元の Mozc 表記に戻す（RevertZenzToMozc）。その一打では次候補へ進まない（`:3443-3461`, `:8439-8479`）。
- **通常の文字で確定したときの Undo**
  - zenz が採用済みの場合は `CommitStringDirectly` を通り、Undo 用の状態を積まない。
  - 応答待ちの場合は `CommitInternal` が一度積むが、次の文字を composer に入れたときに `ClearUndoContext()` が走る（`:9167-9169`）。
  - 採用済みの直接確定では、`zero_query_suggestion` が有効なら Suggest を呼ぶ。応答待ちの確定では呼ばない。そのときに候補窓が実際に見えるかは未検証。
- **§1-8 の「影響しない」**: 当該 state の Tab 割り当てに限った話。
- **Shift+Enter（CommitFirstSuggestion）による先頭サジェストの直接確定**
  - 非ライブの通常サジェストでは残る。
  - 手動 Force 後の CONVERSION では、ライブ専用の条件（設定 on）を満たさないので使えない（astra 指摘・一部確認）。
- **比較を予約する理由の追加分**（どれも key の一致は要求しない）
  - `candidate_selection_after_zenz`（`:4056`）
  - `suggestion_click_after_zenz`（`:4289`）
  - `commit_visible_suggestion_after_zenz`（`:9727`）
  - 予測中の Ctrl+Delete で削除・抑制に成功した場合は、ConvertCancel で比較の予約を破棄する（`:10681-10689`, `:11073-11084`）。
- **Mozc 履歴に学習される単位**
  - 投影した文節の学習件数が 0 のときは、全文の読み・表記で学習するのに加えて、逆変換由来の文節（`MaybeLearnZenzReverseSegmentsToMozcHistory`）でも学習を試す。
  - 通常の確定と互換経路で、同じ順序になっている。
- **比較値の例外**: 取消後の句読点単打確定では、末尾の句読点を除いた表記を先に保存する（`:8774-8788`, `:9182-9192`、astra 指摘）。
- **PASSWORD**
  - 保証できるのは、composer の field type が PASSWORD なら、Force が変換要求を出さないところまで。
  - Mac の Secure Event Input 時に、Context と composer の状態が常に同期しているかは未検証。
- **Mac の描画**（astra 確認）
  - 手動 zenz の出力の `live_conversion` をコントローラが受け取る。
  - 文節の強調・下線の属性は付けない。
  - 非空なら選択範囲を指定しない（`mac/mozc_imk_input_controller.mm:740-744,879-899`）。
  - OS 上の最終的な描画は未検証。
- **Mac の通常文脈**（astra 確認）: 文書長が 1000 単位を超えた場合や、選択範囲が取れない場合にも取得を見送る。
- **auto-block の分子**
  - 分子は通常の rejected の件数で、手動の hard reject は含まない。
  - 短い読みの除外は、実行時の自動再利用に効く。

### 行番号の補正（nits）

| 項目 | 行番号 |
|---|---|
| `live_keymap_active` の式 | `:3229-3231` |
| 既存 zenz 状態のクリア | `:7402` |
| 表記の取り出し | `:7395-7399` |
| 最小読み長を飛ばす箇所 | `:7476-7479`, `:8081-8084` |
| 文脈要求の 4 条件 | `zenz_context_request.cc:25-29` |
| 候補 ID 選択の照合 | `:4012-4053` |

## 未確認のまま残ったこと

- 選択中の予測候補の上で zenz を走らせたとき、validation / adoption が実際にどう判定するか。
- 手で選んだ候補や区切り直した文節が、採用ポリシーで守られるか（ポリシー本体は未読）。
- 中間段階で、zenz の応答が続きを含まないことが保証されているか。最終形に別の validator が必須かどうか。
- 最終形で、確定結果の key、feedback の key、Mozc 履歴をどうするか（未確定の設計論点）。
- Space 変換の順位への feedback の実際の効き方（rewriter は未読）。
- Mac・Linux での、手動 zenz の実際の描画と表示。
- ibus の自前描画が「Tabキーで選択」を出すか。
- テストの件数と中身。
- frontend と GUI の各主張は、私は確認したが astra は未検証。

---

## レビュー反映ログ（round 1）

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| C1 | Shift+Tab / Space は即 rejected ではなく比較の予約。Down は予測経路に入る | 採用 | `:6460-6468`, `:6579-6629` で比較の予約だと確認。ms-ime.tsv:92 で Down=PredictAndConvert を確認。§1-7・§4 を訂正 |
| C2 | Tab の直後に文字を打つと、応答の到着順で確定文字列が変わる。適格性判定が残る | 採用 | `:8904-8935` の 3 分岐と `:7413-7420` を確認。§1-9 を新設し、§0 に追記 |
| C3 | 最終形で何を確定し何を学習するかが未解決。mozc_value は表記の連結。必ず不一致とは言えない | 採用 | `:1664-1679`, `:8497-8513`, `:9816-9835` を確認。§1-5 の断定を撤回し、§4 に節を追加 |
| S1 | Tab と Down の適用範囲。HOLD とスナップショットに限った保証 | 採用 | 全 tsv の Down を grep して表にした（§0）。§1-2 を修正 |
| S2 | 保護スパンはユーザー辞書だけではない | 採用 | `:960-1042` で ASCII・固有名詞・直接辞書を確認。§1-4 を修正 |
| S3 | 予測の上に zenz があると Prediction keymap が優先される。Ctrl+Delete が届かない | 採用 | `:3233-3249` の優先順位と ms-ime.tsv:157-162 を確認。§1-5 に追記 |
| S4 | Backspace・候補操作・Undo の見落とし | 採用（一部は未確認のまま） | Backspace=Cancel の経路と CommitStringDirectly に PushUndo がないことを確認。ピン留め・ID 選択は未読なので未確認と明記 |
| S5 | WithoutFeedback と最小読み長 | 採用 | `zenz_feedback_store.cc:1374-1385` で、記録ではなく判定から除外していることを確認。auto-block の条件を書き直した |
| S6 | defer の期限は 300ms。TakeResult より先に打ち切る | 採用 | `:276-278`, `:1167-1176`, `:7901-7924` を確認 |
| S7 | ライブ off の追加影響、SetConfig は要求を取り消さない | 採用（一部は未確認と明記） | `:10696-10757` は確認。他は astra 指摘として表に入れた |
| S8 | 裏計算と学習（shadow compare、世代は 2 種類） | 採用 | `:8271-8283` を確認。§4・§7-1 を修正 |
| S9 | 空の CUSTOM は既定にフォールバックする。migration は一律追記ではない | 採用 | `keymap.cc:199-206` と config_handler の migration を確認 |
| U | 添付していないファイル由来の「確認済み」を未検証に | 一部採用 | 他の tsv・frontend・GUI・renderer・Linux の IsAvailable は私が実際に読んで確認したので、「確認済み・astra 未検証」と併記した。全面的に未検証へ格下げはしない |
| N1 | 行参照の修正 | 採用 | `:7374-7375`, `:7390-7402`, `:7538-7540`, `:7563-7565`, `:6625-6629` を照合して直した |
| N2 | 既定値の行、インポートメニューの説明 | 採用 | `config_handler.cc:353,356,397` に直し、インポートメニューの説明を補った |

---

## レビュー反映ログ（round 2・旧版向け指摘）

astra は更新版の本文を受け取っておらず、旧版の記述と添付ソースを照合している。そのため、更新版にも当てはまる指摘だけを、私が自分で確かめてから反映した。

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| C(再確認) | round 1 の critical 3 点が保たれているか | 保持を確認 | C1 は §1-7 と §4（比較の予約。accepted / rejected / neutral の分岐）、C2 は §1-9（3 分岐と別経路）、C3 は §4 の「何を確定し、何を学習するか」に残っている |
| S1 | Down の keymap ごとの差。ATOK では Down で最初の文節が確定する | 採用 | atok.tsv:64,118-127、mobile.tsv:92、chromeos.tsv:82 を grep で確認した。§0 に「Down を使えるのは窓があるときだけ」と ATOK の注意を追記 |
| S2 | Windows で候補窓を隠す処理の例外と、ruby window の実表示 | 採用 | `window_manager.cc:247-288` の passive SUGGESTION の例外を確認した。§3 で、分岐（確認済み）と実際の表示（未検証）を分けた |
| S3 | Mac / Windows の前後文脈の条件 | 採用 | `mac/mozc_imk_input_controller.mm:518-520,1444-1458` を確認した。live off では TEST_SEND_KEY を省き、通常の文脈は入力開始時などに限られる。§3 に追記 |
| S4 | 「frontend は Tab を横取りしない」の範囲 | 採用 | Mac の `keyCodeMap_` への委譲と早期 return（`:1398-1425`）を確認した。§5 の結論を「調べた範囲では見当たらない」に限定した |
| S5 | Qt の Shift+Tab | 採用 | `keybinding_editor.cc:354-375,566` を確認した。:566 はひらがな／カタカナ専用で、修飾は `shift_key_name_` 由来（LeftShift / RightShift になりうる）。§6 を訂正 |
| N1 | ibus の Shift の付け方 | 採用（引用のみ） | `:348-362` は私は未読で、astra の確認として引用した。結論は変わらない |

---

## レビュー反映ログ（round 2 正式版）

critical はなし。should-fix と nits は、すべて自分でコードを確かめてから判断した。

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| S1 | 手動 Tab の次の文字で何が確定するかは、「session が採用・表示済みか」で決まる。ライブ off の手動 Tab では deferred は起きない | 採用 | Force が `ClearZenzLiveCorrectionState` で deferred を消すこと、deferred を作るのは自動ライブだけであることを確認した。§1-9 に追記 |
| S2 | 自動 block の境界（accepted 1 件があっても、rejected 1 件で 50% に達する） | 採用 | `zenz_feedback_store.cc:642-659` の `rejected*100 >= pct*obs` を確認した。§4 の誤りを訂正 |
| S3 | 比較するのは確定した表記。分割確定の扱い。Mozc 履歴は文節単位が先 | 採用 | `:6490-6502` と ConfirmPending の互換経路を確認した。§4 の「当然 rejected」を条件付きに直した |
| S4 | chromeos / mobile の Composition Down は MoveCursorToEnd。「窓があるときだけ」は 5 種共通の範囲に限る | 採用 | 全 tsv の grep で確認した。§0 の表と注記を修正 |
| S5 | コマンドを AI 化する場合の入口は 2 つ。CUSTOM の Precomposition。表示リストの検証条件 | 採用 | `:3480-3498`、`:11093-11132`、`:2795-2798` を確認した。§1-8 / §1-8b を追加 |
| S6 | Ctrl+Delete は裏の候補を削除する。ピン留めは拒否。ID 選択には整合が必要。Enter による zenz 確定は Undo を積まない | 採用 | `:10681-10689`、`:4090-4099`、`:4224-4229`、`:3385-3402` を確認した。§1-10 の未確認を解消 |
| S7 | SetConfig の本体は `:3771-3775`。英数 Space 分岐は live 設定と無関係 | 採用 | SetConfig は ClearUndoContext と設定の差し替えだけだと確認した。§3 の表を訂正 |
| S8 | 世代は 2 種類ではなく、遅延ライブ変換の世代もある | 採用 | `:4374-4384` の `live_conversion_generation_` を確認した。§7-1 を修正 |
| U | 添付していない実装に依存する主張は未検証として扱う | 採用 | 凡例を改め、対象を列挙した。round 2 で astra が確認した範囲は除いた |
| N1 | 行参照の補正 | 一部採用 | Mac の 20 単位の上限は astra の確認として引用した。その他の細かい行の補正は本文の結論に影響しないので、全部は反映していない |
| N2 | Windows の passive SUGGESTION は「隠さない」ではなく「通常の描画へ進む」 | 採用 | §3 の文言を直した |
| N3 | Mac の通常の文脈は左側だけで、右は設定されない | 採用 | `:1211-1231` を確認した。§3 に追記 |
| N4 | Qt の Shift+Tab と IBus の確認 | 採用 | 未確認の一覧から「Shift+Tab の組み立て」を削除した |

---

## レビュー反映ログ（round 3）

critical はなし。指摘は自分で確かめてから判断した。**レビュー回数の上限（3 回）に達したため、この round 3 の反映分は astra の再確認を受けていない。**

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| S1 | 窓がある・同じ候補を選べる・予測に入れる、は別の条件。§1-1 の「手段がなくなる」は過大 | 採用 | §1-1 を「一打で入る既定キーがなくなる」に直し、Down の経路を追記した |
| S2 | 応答待ちの Space、ライブ由来の場合だけ Backspace が一字編集になること | 採用 | `:3443-3461` と `live_keymap_active` の条件を確認した。§1-10 と §8 に反映 |
| S3 | 通常文字での確定の Undo とゼロクエリ | 採用 | `ClearUndoContext()`（`:9169`）を確認した。未確認の一覧から外した |
| S4 | Shift+Enter と、比較を予約する理由の追加分 | 採用 | 予約理由の 3 つを grep で確認した。CommitFirstSuggestion のライブ条件は astra 指摘として一部確認 |
| S5 | 学習単位（逆変換文節）と、句読点の例外 | 採用 | 互換経路で `MaybeLearnZenzReverseSegmentsToMozcHistory` を確認した。句読点の例外は astra 指摘として記載 |
| S6 | Force の意味は開始状態で変わる。採用判定は残る。PASSWORD の範囲 | 採用 | §7-2 を書き直した。採用判定の具体例は astra 指摘として記載 |
| S7 | 確認範囲の過大・過小 | 採用 | 凡例どおり未検証の対象を維持した。Mac の描画の確認範囲を §8 に追記 |
| N1 | 行番号の補正 | 採用 | §8 に表としてまとめた |
| N2 | 期限の境界（「以上」、3000ms 側は TakeResult の後） | 採用 | §3 の表の文言を直した |
| N3 | auto-block の分子、短い読みの除外が効く範囲 | 採用 | §8 に追記 |
| N4 | Mac の文脈取得の見送り条件、IBus の pending 条件 | 採用 | §8 に追記。IBus の pending は §3 で既に記載済み |
