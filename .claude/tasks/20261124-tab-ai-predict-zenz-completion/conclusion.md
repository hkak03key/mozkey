# Tab の「AI 予測変換」に zenz を使えるかの調査結論（review round 1〜3 反映版）

## 要約（ユーザー向け）
- **結論**: 試作には進める。製品に採用するか、候補を何位に出すかは、実測してから決める。
- **方式**: zenz に、打った読みの続き（カタカナ）を予測させる。その読みを足した全読みを変換して、予測候補にする（2 段方式）。azooKey に実験実装があり、確認済み。変換出力に直接続きを書かせる方式は、学習目的から外れていて、裏付けになる評価も無い。
- **mozkey 側**:
  - scorer（Windows・POSIX とも）は、予測用の prompt をそのまま送れる。確認済み。
  - ただし scorer は、返ってきた読みが正しいことを保証しない。補完が無かった場合と、途中で打ち切られた場合も区別できない。
  - Tab 側では、表示済みの候補を昇格する経路と、学習の入口に、新しい設計が要る（推測）。
- **既存の live correction 経路にそのまま流すのは不適切**。確認済み。
- **未確認の主なもの**:
  - 読み補完の的中率。公開資料には無い。
  - Tab を押してから AI 候補を選べるまでの時間。
  - 既存の正解を押し下げる率。
  - 配布物（バイナリ・重み）のハッシュ照合。

- 上流 azooKey の照合版: レビュアーが確認した版は `fddde10d346ed13b4ae0058327be38e8b26c8900`。調査者が最初に読んだのは default branch の HEAD で、SHA を控えていないため、同じ版かは確定できない。
- 「確認済み(自)」は、調査者がその場でファイルを読んだという意味。各 round のレビュアーにそのファイルが添付されていたかとは別。

- 取得日: 2026-11-24（web・GitHub の取得はすべてこの日）
- azooKey のコード: default branch の HEAD を gh-explorer で読んだ。レビュアーが固定した版 `fddde10d…` と同じ SHA かは控えていない。
- 調査範囲: read-only。コードは変更していない。
- 確度の書き方
  - **確認済み(自)**: 自分でコードや URL を読んで確かめた
  - **確認済み(レビュー)**: レビュアーが示した根拠を自分で開いて確かめた
  - **レビュー経由・未検証**: レビュアーの指摘だが、自分では開いていない
  - **推測**: 推測

## 要約

**判定: 試作には進める。採用の可否を決めるには根拠が足りない。** 遅延と品質の実測、学習経路の設計が、まだ無い。

| 問い | 結論 | 確度 |
|---|---|---|
| 1. zenz は「読みの続きの補完」に使えるか | **条件付きでできる**。「読み（カタカナ）の続きを予測し、伸ばした読みを変換する」2 段方式には、azooKey の実験実装という先例がある。変換出力に直接続きを書かせる方式は学習目的から外れ、実用性を示す評価も無い | 先例の実装: 確認済み(自)。品質: 未測定 |
| 2. mozkey の zenz 経路で扱えるか | scorer と wire protocol は**予測用 prompt を送るところまでは**そのまま使える。返ってくる文字列がカタカナとして正しい続きである保証は無い。session 側には「予測候補」として扱う新しい経路が要る。今の live correction 経路にそのまま流すのは不適切 | 確認済み(自)。設計案は推測 |
| 3. 遅延 | **実測が必要**。既存の 180ms は live correction 1 要求の timeout で、Tab の予算ではない | 定数: 確認済み(自) |

---

## 1. zenz モデル側

### 1-1. 学習形式（確認済み(自)）
- azooKey/AzooKeyKanaKanjiConverter `Docs/zenzai.md`
  - v1: `<input_katakana><output></s>`
  - v2: `<input><context><output></s>`
  - v3: `<context><input><output></s>` を推奨。`` profile、`` topic、`` style、`` 設定
  - v3.2: `` 右文脈
- model card
  - https://huggingface.co/Miwa-Keita/zenz-v2.5-small : GPT-2、91M、文字単位＋バイト BPE、"条件付き言語モデル"、CC-BY-SA 4.0
  - https://huggingface.co/Miwa-Keita/zenz-v3.1-small と https://huggingface.co/Miwa-Keita/zenz-v3.1-small-gguf : README は空。90.5M / 95.1M、Q5_K_M は 73.9MB
- 著者論文 https://www.anlp.jp/proceedings/annual_meeting/2025/pdf_dir/P1-19.pdf
  - 「入力と出力をつないだ列に次トークン損失をかける」「ニューラル単体では入力への忠実性を保証しにくい」（**レビュー経由・未検証**。PDF は自分で読んでいない）
- 論文の Acc@1・CER は「与えた読みの変換」の精度。未入力部分の的中率の根拠には使えない（レビュー経由・未検証）。

### 1-2. 変換出力（`` の後ろ）に直接続きを書かせる方式（**修正**）
- 旧: 「不可（根拠は強い）」
- 新: **学習目的から外れ、実用性を示す評価が無い**
  - 学習課題は「与えた読み全体を表記に変換して EOS」（1-1 の形式）。読みの途中で `` を置いたとき、続きまで書く根拠は弱い（推測）。
  - 一方、mozkey の生成設定（greedy、``〜`` と改行で stop）には、出力の読みを入力の長さに縛る仕組みは無い（確認済み(自): src/zenz_scorer/llama_http_client.cc:688-706）。長く出ること自体は起こりうる。
  - 採用候補としては 2 段方式を優先する。

### 1-3. 読みの続きを予測する方式（azooKey の実装。確認済み(自)。細部はレビューを受けて訂正）
- prompt（`ZenzPromptBuilder.swift` の `inputPredictionPrompt`）
  - 並び: v3 条件（EE03〜06）→ `[EE02 左文脈]` → `[EE07 右文脈]` → `EE00 読み`。**`EE01` は付けない**
  - 空の左文脈・右文脈はタグごと省く
  - v3 以外では nil を返す
- 生成（`ZenzInputTextGenerator.swift`）: 1 文字ずつ自前のループで回す。各 step で次を行う。
  1. **全語彙**の logit に、位置で減衰する反復補正（`1 + Σ 2/距離` で割る）をかける
  2. 補正後の全語彙から entropy を計算する
  3. 有効な UTF-8 に復号できる token の、先頭 1 文字を候補にする
  4. 生成済みの続き全体が `possibleNexts` のどれかの prefix になっている候補だけを残す
  5. 最大値を選ぶ
  6. その 1 文字を再 tokenize して prompt に追加する
- 停止条件
  - 生成済みが `minLength` 文字以上で、かつ entropy が `maxEntropy` 以上、または句読点「、。！？」が出たとき
  - **`minLength` は「1 文字は必ず返す」保証ではない**。entropy・句読点で止める判定を始める条件にすぎない（レビュー指摘。コードで確認済み(自)）
- 未確定ローマ字の扱い（`Prediction.swift` の `resolvePredictiveInputSource`。確認済み(レビュー)）
  - 例: ローマ字入力で末尾が「ありがt」
  - 末尾のローマ字列を取り除いた `baseConvertTarget`（「ありが」）をモデルに渡す
  - 入力表から `possibleNexts`（t から始まりうる「た」「ち」「つ」…）を得る
  - `droppedSuffixCount` を返し、変換用のコピーでは元の未確定末尾を削除してから予測を挿入する
- 使われ方（`KanaKanjiConverter.swift` の `getPredictionCandidate`。確認済み(自)）
  1. まず辞書の予測を作る。`prefixText` が指定されていれば、表記の prefix で絞る
  2. **絞った後に辞書の予測が空**で、かつ zenzai と `experimentalZenzaiPredictiveInput` の両方が有効なら、zenz の予測に進む
  3. `count: 10, minLength: 1, maxEntropy: 3.0` で予測する
  4. scratch session で、日本語・英語の予測を無効にして通常変換する
  5. 第 1 候補を返す。第 1 候補が `prefixText` に合わなければ 0 件
- cache（`predictiveInputCache`）
  - 文脈・入力方式・重み・version 設定が一致し、予測の未消費部分が残る間は、**読みの生成だけ**を省く
  - 後段の変換は毎回走る
- 既定状態（**レビュー経由・未検証**）: ライブラリ引数の既定は `false`。azooKey-Desktop は明示的に `true` を渡すが、利用者向けの `DebugPredictiveTyping` は既定 `false`。したがって macOS 版の利用者向けでは既定オフ。
- 関連の試作: 開発者による「次の 1 文字の情報を予測変換の並べ替えに使う」PR #305（https://github.com/azooKey/AzooKeyKanaKanjiConverter/pull/305 ）。draft のまま未マージで close。本文は「『次入力予測』を Zenz が学習していることを利用し」（確認済み(レビュー)）。開発者自身が、zenz が次入力予測を学習していると述べている。

### 1-4. mozkey で取りうる prompt と生成方法（推測。レビューを受けて訂正）
- 段 1（読みの予測）: 上流を再現するなら、**上流の並び（条件が先頭、空の文脈はタグを省く）**に固定する。
  - mozkey の builder は今、次のように組み立てている（確認済み(自): src/session/zenz_prompt_builder.cc:228-253）。
    - 左文脈が空でも `EE02` を置く
    - 条件（EE03〜06）を左右文脈の後ろに置く
    - 末尾に必ず `EE01` を付ける
  - 並びの差が結果に効くかは未測定。比較試験では、この差を固定して測る。
  - 条件文字列の切り方も違う。上流は**末尾 25 文字**（`suffix(25)`）を取り、mozkey は**先頭から最大 64 文字**（`kMaxZenzConditionChars`、zenz_prompt_builder.cc:20, 113-146）を取る。長い profile などで比べるときは、モデルに実際に渡る文字列を揃える。
- 段 2（変換）: 「打った読み（未確定末尾は除く）＋予測した読み」を、通常の変換 prompt で zenz に送るか、Mozc で変換する。Mozc で変換すれば、zenz の呼び出しは 1 回で済む。
- 上流を完全に再現するには、生成ループの中での判定が要る（全語彙の entropy、距離別の反復補正、possibleNexts の制約）。
  - llama-server の `n_probs`（生成位置の上位 N だけ）や、標準の `repeat_penalty` では正確に再現できない。
  - この判定を scorer の中で完結させれば、確率を session まで返す必要は無い（レビュー指摘。妥当と判断）。
  - 簡易版の案（推測）: grammar（GBNF）でカタカナと possibleNexts に制約し、固定の n_predict で止める。

## 2. mozkey の zenz 経路

### 2-1. 再利用できる部分と、その限界（確認済み(自)）
- **wire protocol と scorer は、変換用か読み予測用かを区別しない。prompt 末尾のタグも検査しない。**
  - 要求ヘッダは magic、version、kind、generation、timeout_msec、max_output_chars、prompt_size（src/zenz/zenz_wire_protocol.h:26-34）
  - POSIX server はヘッダの値と prompt サイズ（8192 bytes 以下）を検査し、llama-server `/completion` にそのまま渡す（src/zenz_scorer/posix_server.cc:457-483、llama_http_client.cc:688-706, 741-772）
  - したがって `EE01` を付けない予測用 prompt も送れる。
  - **「そのまま使える」の範囲は通信経路（prompt を送り、整形された文字列を受け取る）に限る。** Windows も同じ（確認済み(自)）。
    - Windows の scorer（main.cc:1190-1229）も greedy、非ストリーム、固定の stop で要求する。
    - 応答の処理（main.cc:1309-1323）は `CleanGeneratedText` で整形し、空なら `empty_content` を返す。
    - 読みとして妥当か、azooKey 相当の停止判定、補完の品質は、どちらの OS でも scorer は保証しない。
- **ただし、返り値が有効なカタカナの続きである保証は無い**（critical 1 を採用）
  - 返った `content` は、最初の stop marker（EE00〜EE0F、`</s>`、改行など）で切られ、空白を除かれ、`max_output_chars` で切り詰められるだけ（llama_http_client.cc:668-687）。
  - 整形した結果が空なら `empty_content` エラーになる（llama_http_client.cc:931-934、Windows は main.cc:1317-1320）。
    - 診断文字列が違うので、通信エラーとは区別できる。
    - しかし、空になる原因は区別できない。先頭に `EE01` が出た、別の停止記号が出た、空白だけが出た、のどれでも同じ `empty_content` になる。
    - 非空なら、token の上限で切れても `ok` になる。
    - したがって**今の wire の応答からは「先頭 `EE01` 率」も「途中打ち切り率」も測れない**。
    - 測るには、scorer が整形する前に、llama-server の応答の `stop_type` と `stopping_word`、`max_output_chars` による切り詰めの有無を、別々に数える必要がある（llama.cpp 620d2225 版の README の応答仕様。レビュー経由・未検証）。
  - 区切りが出ないまま n_predict に達すると、途中で切れた文字列が ok として返る。
  - カタカナかどうかを確かめる検査も無い。
  - 試作では「有効な読みの続き」「補完なし」「通信・推論の失敗」を区別する必要がある。
- n_predict = `max(4, min(設定 n_predict, max_output_chars))`
  - 設定 n_predict は既定 64。POSIX では 4〜256（posix_runtime.cc:19-20, 46-48）、Windows では環境変数 `MOZC_ZENZ_N_PREDICT` で上限 256（main.cc:65, 87, 311-312, 1190-1194）。
  - session は max_output_chars=256 を送る（session.cc:7856）。
  - したがって既定構成では 64 token（確認済み(自)。環境変数・設定での上書きは除く）。
  - max_output_chars は、生成 token の上限と、返却文字列の切り詰めの両方に効く。
- 既定のモデルと ctx
  - POSIX: `zenz-v3.2-small-Q5_K_M.gguf`、ctx 256（main_posix.cc:17-20）
  - Windows: ctx 256（main.cc:62）。モデルは、既定で実行ファイルの隣の `models/zenz-v3.2-small-Q5_K_M.gguf`。モデルのパスを環境変数 `MOZC_ZENZ_MODEL` で上書きできるのはデバッグビルドだけ（main.cc:291-297）。
  - 起動待ちは最低 1500ms（main.cc:73）
  - （確認済み(自)。実際に配布物へ同梱されている重みは未確認）

### 2-2. 今の live correction 経路に補完値をそのまま流した場合（**訂正**）
例: key=ありが、value=ありがとうございます

| 段 | 振る舞い | 根拠・確度 |
|---|---|---|
| `Validate` | この例の場合: 長さ検査（`key_len*3+8`）は通る。3 文字なので最小長 2 も満たし、表記は URL・secret の判定にも当たらない。一般には、ほかに `same_as_mozc`、`too_short_key`（session からは既定 2 が入る）、`left_context_echo`、URL・secret、`synthetic_disabled`（`use_zenz_synthetic_candidate` が OFF なら）などで拒否されうる。**この例だけで「通る」とは断定できない** | zenz_output_validator.cc:913-977、session.cc:275, 1178-1183, 7206。確認済み(自) |
| strict reading preservation（既定 OFF） | ON なら、逆変換した読みが key と一致するか、正規化した key の読みと一致したときだけ許可する。この例では、拒否理由は次のどちらかになる。`reading_mismatch` は、逆変換で読みが取れて、かつ元の key の正規化読みが空でないとき。`reading_unavailable` は、逆変換で読みが取れないとき、または元の key の正規化読みが空のとき | session.cc:191-229、config.proto:455、zenz_output_validator.cc:896-911。確認済み(自) |
| adoption | 保護語や表記の修復、segment への投影を行う。key と value の例だけでは結果が決まらない | zenz_adoption_policy.cc:1300-1388。確認済み(自)。結果は実行しないと不明 |
| 表示 | 採用されると、preedit を Clear して、強調表示の segment を 1 つ作る。その後、未確定ローマ字の末尾を表示する処理を通るので、最終的な表示が常に 1 segment とは限らない。strict が有効なら、表示の前に、修復後の表記を元の key に対して検査する。**表示だけでは accepted は記録しない**（コメント: "Displaying a zenz correction is not the same as user acceptance"）。記録はユーザーが確定したとき | session.cc:8214-8227, 8361-8388。確認済み(自) |
| feedback の**再利用** | session の fast path は**複数 segment の live conversion のときだけ**使われる（session.cc:7097-7106）。単一 segment のときは converter 側の `ZenzFeedbackCandidateRewriter` が扱う（そちらが同じ閾値を使うかは未確認）。store は key を**完全一致**で引く（zenz_feedback_store.cc:1426-1440）。**訂正**: fast path は `GetAcceptedCandidates(key, ctx, GetZenzFeedbackReusePolicy(config))` を使う。`use_zenz_feedback_min_key_length` は既定 true、`zenz_feedback_min_key_length` は既定 **7**。key が 7 文字未満なら空を返す。「ありが」（3 文字）は再利用されない。7 文字以上の key（例: 「ありがとうござ」）では再利用されうる。incognito のときは再利用しない（session.cc:1209-1212） | config.proto:461-467、config_handler.cc:390-394、session.cc:1209-1212, 1232-1246, 7097-7106, 7157-7160、zenz_feedback_store.cc:37-47, 1426-1440。確認済み(自) |
| feedback の**記録** | 閾値は記録には効かない（proto のコメント: "Recording and Mozc history are not affected"）。記録されるのは、`use_zenz_feedback_recording` が有効で、**かつ** incognito でなく、private input でなく、パスワード欄でもなく（`CanPersistZenzLearningFromInput`）、さらに key と value の privacy 検査を通ったとき。その場合は `RecordAccepted(key, ctx, value)` で、短い key と長い value の対が記録される | session.cc:1203-1225, 5539-5586。確認済み(自) |
| Mozc 履歴の学習 | **訂正**: 確定時に `CanLearnZenzToMozcHistory`（`use_zenz_mozc_history_learning` とプライバシーの条件）を確かめ直してから、まず投影した区間で学習する。**投影した区間での学習が 0 件なら、保存してある全体の key/value での学習（`MaybeLearnZenzCandidateToMozcHistory`）と reverse segment での学習にフォールバックする**。したがって「投影に失敗すれば誤学習も起きない」とはいえない。逆に「表示した時点で誤学習する」ともいえない。成立するのは、確定し、かつ学習が許可されたときだけ | session.cc:637-680, 1227-1230, 5786-5821。確認済み(自) |
| worker | **訂正**: `Submit` が上書きするのは**待機中**の要求だけ。実行中の `Convert` は完了し、session が `TakeResult` で generation を照合して古い結果を捨てる。**同じ worker を共有する設計**では、待機中の要求や未回収の結果が上書きされうる。**ただし、worker を分けても推論の待ちは解消しない**。Windows は llama-server を `--parallel 1` で起動し、pipe の `HandleClient` も 1 件ずつ処理する。POSIX も `ServeOne` の中で backend を同期的に呼ぶ。そのため、live correction の推論中に Tab が来ると、その推論が終わるまで待つ | zenz_live_corrector.cc:53-125、main.cc:1074-1077, 1792、posix_server.cc:391-501。確認済み(自) |

→ 結論は変わらない: 補完は live correction の値ではなく、予測候補として別の経路で扱うべき（推測。設計判断）。

### 2-3. 扱えるようにするために変えるもの（推測。レビューを受けて訂正）
1. **候補の表現**: 次の 3 つを区別して持つ。
   - 打鍵済みで確定した読み（未確定のローマ字末尾を除く）
   - 置き換える未確定末尾
   - 予測後の全読み
   - 候補は key = 全読み、value = 変換にする。
   - 検証は単純な `starts_with(生の入力)` ではいけない。「未確定末尾を除いた読みが prefix で、かつ続きが possibleNexts と合う」で判定する。生の「ありがt」に対して単純な prefix 判定をかけると、正しい「た…」も落ちる。
   - 「合う」の定義を決める必要がある。上流は、`possibleNexts` が空でない間は、**生成した続き全体**が表のどれかの要素の prefix であり続けることを求める。たとえば表が `["タ"]` だけなら、「タ」は返せるが「タヨ」は返せない。表が空ならこの制約はかからず、上流には一律のカタカナ限定の検査も無い（確認済み(自): ZenzInputTextGenerator.swift の isAllowedPrefix）。
   - 上流と同じ条件にするか、最初のかなが決まった後は自由に延ばすかで、比較する方式が変わる。
   - GBNF を使う場合は、今の scorer の固定 JSON（llama_http_client.cc:688-706）に `grammar` を足す必要がある。
   - **再利用の入口も決める必要がある**: 全読み「ありがとうございます」で保存しても、store は完全一致で引くので、次に「ありが」で引いても見つからない。再生成した候補を全読みで照合するのか、打鍵済みの prefix で候補を検索するのかを決める。
2. **prompt builder**: `EE01` を付けない予測用の builder を作る。並びは上流と同じにする。
3. **scorer**:
   - 最小構成: 今のままで送れる。ただし `empty_content` には「補完なし」と「出力の異常」が混ざり、「途中打ち切り」は `ok` に紛れる。停止の理由（stop_type）を返すこと、カタカナであることの検証が要る。
   - 上流の再現: 生成ループを scorer 内に持つ（1-4 参照）。wire version を上げる必要がありうる。
4. **session の経路**
   - Tab は MS-IME の keymap で、Composition 時も Conversion 時も `PredictAndConvert` に割り当てられている（data/keymap/ms-ime.tsv:60, 128）。
   - **同期の `Predict()` に入る前に、通常経路（`PredictAndConvert`, session.cc:11135-11164）でも live conversion 経路（`PredictAndConvertFromLiveConversion`, session.cc:11093-11133）でも、先に `FocusDisplayedPassiveSuggestion()` を試す**（session.cc:10070-）。
     - 表示済みのサジェストを昇格できれば、再予測せずに終わる。
     - 通常経路で、すでに CONVERSION 状態なら `ConvertNext()` に進む。
     - live conversion 経路では、条件を満たさなければ `PredictAndConvert` に戻る。
     - （確認済み(自)）
   - **未確定ローマ字は、生成するときだけでなく、Tab で選ぶ時点でも照合する**。
     - 確定したかなの後ろに未確定ローマ字が来ると、mozkey は直前のサジェストを保持することがある（`ShouldHoldSuggestionForPendingRoman`、session.cc:4445-4462, 9005-9039）。
     - Tab では、保持した候補 ID を再予測せずに昇格できる（session.cc:10070-）。
     - 例: 「ありが」で作った AI 候補が、「ありがt」でも同じ ID と表記のまま残る。このとき、補完が `t` に合っている保証は無い。
     - したがって、保持と昇格の経路でも、候補が表す全読み・置き換える末尾・今の入力表の 3 つが整合しているかを検証する。
     - 上流も、入力表に該当する `possibleNexts` が無いローマ字の末尾は削除しない（確認済み(自): romanSuffixAndPossibleNexts）。
     - （session の該当箇所は確認済み(自)。設計案は推測）
   - **送信前の入力保護も引き継ぐ**。今の経路は、パスワード欄・key・Mozc の値の privacy を検査し、左右の文脈を組み立ててから prompt を作る（session.cc:7446-7508。確認済み(自)）。新しい予測経路でも同じ検査が要る。
   - したがって、AI 候補を `Predict()` の直後にだけ差し込むと、表示済みの候補を昇格する経路を取りこぼす。**候補窓に表示した候補と、Tab で実際に選ばれて確定される候補が一致する**ところまで設計する必要がある。例えば、AI 候補をサジェストの時点で候補窓に入れておき、昇格の経路でも同じリストを使う（推測。設計案）。
   - 取りうる方法: 同期で待つ（短い timeout）／サジェスト時に先行して計算して cache／候補窓を後から更新。
   - live correction とは別の worker か、優先度の制御を用意する。
5. **validator**: 予測用に別の基準を設ける（prefix と possibleNexts の整合、全読みの逆変換、長さ上限）。
6. **学習**: 記録・再利用・Mozc 履歴の 3 つそれぞれで、key を全読みにする。

### 2-4. 案の比較材料（推測。レビューを受けて拡充）
| | A: Mozc の予測を zenz で並べ替える | B: 2 段生成のフォールバック（上流と同じ） | C: 2 段生成の結果を上位に混ぜる |
|---|---|---|---|
| zenz に要るもの | 候補の系列尤度（厳密に求める場合）、**または**次の読み 1 文字のスコアを Mozc の順位に足す（PR #305 方式） | 今の `/completion` で送れる。品質のためには生成ループの改修 | B と同じ |
| 制約 | 厳密な尤度は、各候補の先行列で各 token を採点する必要がある。目的の token が top-N に入らなければ確率が取れない。mozkey の runtime lock は llama.cpp の commit `620d2225cc436ab0b001e3200e590e3b286ef9ab` を固定している（src/zenz_runtime/llama_runtime.lock.json:25, 48。確認済み(自)）。その版の README で `n_probs` は**生成位置**の上位 N と定義されている（レビュー経由・未検証）。issue #27174 は別の版の `/v1/completions` の報告。今の scorer は確率を返さない。配布済みバイナリでの実行結果は未確認 | 辞書の予測が空のときだけ出る | **上流とは違う順位付けの実験**になる。誤った補完が上位に来るリスクがある |
| 安全性 | 値は Mozc 由来 | 予測した読みが誤りうる。段 2 で検証する | B と同じ |

- 出典: llama.cpp server README https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md （確認済み(自)）、issue https://github.com/ggml-org/llama.cpp/issues/27174 （確認済み(レビュー)）、PR #305（確認済み(レビュー)）

## 3. 遅延（**実測が必要**。旧版の「約 20 token、数十〜百数十 ms」という見積もりは撤回）

- 既存の時間設定（確認済み(自)）。**いずれも live correction の各区間の設定値で、Tab から表示までの実測値ではない。**

| 設定 | どの区間の値か | 値 | 根拠 |
|---|---|---|---|
| live correction の timeout | 1 要求について、session が scorer に渡す推論の期限（wire の `timeout_msec`） | 未設定なら 180ms、上限 1000ms | config.proto:410、session.cc:273, 281, 1159-1165, 7855 |
| debounce | live conversion を計算してから、zenz に要求を出すまでの待ち | 未設定なら 700ms、上限 5000ms | config.proto:393、session.cc:272, 280, 1151-1157 |
| deferred presentation の timeout | deferred 表示が有効なとき、要求を出してから Mozc の結果を隠しておける期限 | 未設定なら 300ms。明示的に 250ms を保存していれば 250ms。範囲は 50〜3000ms | config.proto:405-406、session.cc:276-278, 1167-1176 |
| poll 間隔 | session が async worker の結果を見に行く間隔 | 24ms | session.cc:274 |
| 非同期の待機窓 | 結果を取れなかった poll のたびに判定する待機窓（cold start を含む）。poll の回数による打ち切りもある。届いた結果を締め出す絶対期限ではない（deferred が無効のとき） | `max(timeout, 3000ms)` | session.cc:288, 7935-7947 |
| scorer の HTTP timeout（POSIX） | scorer から llama-server への 1 回の HTTP の期限 | 50〜30000ms。HTTP client の既定 5000ms は、直接渡された値が 0 のときだけ使われる。wire で 0 が来た場合は、その手前で server が 30000ms に置き換える。通常は session の 180ms が使われる | llama_http_client.cc:33-35, 758-760、posix_server.cc:48, 467-470 |
| Windows の起動待ち | llama-server の準備ができるのを待つ予算の**下限** | 1500ms。準備ができれば、すぐに戻る | main.cc:73, 1521-1558 |

  - 判定の順序: deferred が有効なときは、`TakeResult()` より**先に**表示期限を判定し、期限を過ぎていれば要求を破棄する（session.cc:7901-7917）。そのため、結果が届いていても期限後なら使われない。
  - 測定の材料
    - session の `[zenz] async response` ログ（session.cc:7919-7930）は、ok、timeout、generation などを出すが、**latency は出さない**。
    - 処理時間は、scorer の応答ヘッダの `latency_msec` と scorer 側のログで取れる（Windows は main.cc:1698-1711）。
    - Tab の体感遅延を測るには、候補を表示した側に終点の計時を足す必要がある。
  - 注（deferred presentation の食い違い）
    - proto の既定は **250ms**（config.proto:405-406）
    - session は `has_zenz_deferred_presentation_timeout_msec()` が false なら定数 **300ms** を使う（session.cc:276, 1167-1176）
    - ConfigHandler はこの field を補完しない（config_handler.cc を grep しても該当なし）
    - したがって field が未設定のときの実効値は 300ms で、proto の default 250 は使われない。旧版の「既定 300ms」は実効値としては正しいが、proto の値との食い違いを書いていなかった。
- 180ms は Tab の予算ではない。段 1 の上限は 10 文字だが、段 2 は全文を変換するので、その token 数は全文の長さで決まる。段 2 を Mozc で行えば、zenz の呼び出しは 1 回で済む。`cache_prompt: true` で実際にどれだけ再利用されるかも未測定。
- **測るべき指標**（対象の機械で、初回とウォーム時に分けて測る）
  - 遅延: Tab 押下から候補表示までの**中央値・p95**、**timeout 率**。内訳として scorer の `latency_msec`（zenz_wire_protocol.h:42）と、段 1 と段 2 それぞれの時間
  - 品質: 辞書の予測が**ある場合とない場合に分けて**測る
    - **的中率**: 正しい続きを出した割合（ユーザーが実際に打った続きとの一致）
    - **（訂正）空振り率は 1 つにまとめない**。まとめると、正しく提示を控えた B と、既存の正解を押し下げた C の差が見えなくなる。同じ入力と文脈で、Mozc 単独・B・C を比べ、次の 5 つを分けて測る。
      1. B の発火率
      2. 提示したときの読みの的中率
      3. 最終表記の第 1 候補の正解率
      4. 既存の正解を押し下げた率
      5. 推論の失敗率
    - **（round 3 で訂正）遅延と失敗率の分母を決める**。B では、AI を発火させずに既存の候補をすぐ選べることがあり、その場合「AI 候補を選択できる時点」は存在しない。そこで次のように分ける。
      - 対象にする Tab 全体: 「何らかの候補を選べるまで」の中央値と p95
      - AI が発火した Tab: 「決めた期限内に AI 候補を選べた割合」と、提示できたときの遅延
      - 結果の区別: 辞書の候補があったので発火しなかった／正常な「補完なし」／推論の失敗／期限超過
    - **読みの的中の定義**: 「予測した読みが、実際の続きの先頭と一致した」とする。補完した文字数も併記する。こうすると、1 文字だけ出す方式との違いも残る。
    - **B の発火条件を、mozkey で判定できる形に固定する**。
      - 上流が空かどうかを調べるのは、今回生成した辞書予測を `prefixText` で絞ったもの。外側では、以前の入力から引き継げる候補を、新しい候補より先に最大 3 件まとめる（レビュー経由・未検証: KanaKanjiConverter.swift L1042-1069）。
      - したがって、B が上流と同じなのは「読みの生成を呼ぶ条件」までになる。
      - mozkey では、入力どおりの変換候補が 1 件あるだけで止めるのか、入力より読みが伸びる候補があるときに止めるのかで、発火率が変わる。
      - 比べるときは、辞書・履歴・保持中の候補の扱いを固定する。そのうえで「生成を試みた件数」と「AI 候補を実際に選べた件数」を分けて数える。
    - **先頭 `EE01` 率の数え方**: `stop_type=word` かつ `stopping_word=EE01` で、かつ**空白を除く前の `content` が空**のものだけを数える（レビュー経由・未検証: llama.cpp 620d2225 版の README）。`stop_type=limit`、ctx 超過による `truncated`、scorer での文字数の切り詰めは、別の事象として数える。grammar でカタカナを強制する場合は、制約なしのときの先頭停止率と分けて集計する。
    - **`accepted` の記録を「Tab で AI 候補を選んだ証拠」にしない**。通常の Mozc の確定でも、保留中の比較対象と最終表記が一致すれば `RecordAccepted` が呼ばれる（その分岐では Mozc 履歴の追加学習はしない。session.cc:6579-6614。確認済み(自)）。明示的に選んだ率を測るなら、次の 3 つを区別する。
      - Tab でのフォーカス
      - AI 候補の確定
      - 通常の確定が、後から AI 候補と一致したもの
    - 遅延の終点は、AI が発火した場合は「AI 候補を実際に選択できる時点」にする。
    - 遅延試験には「live correction の実行中に Tab を押す」場合を含め、送信前の待ち時間も測る（2-2 の worker の行を参照）。
    - timeout 率は wire の `Timeout` の件数だけでは測れない。Windows は準備待ちの失敗を `Timeout`、HTTP の失敗を `Error` で返す（main.cc:1664-1671）。また deferred が無効なら、async の待機窓は `TakeResult` が空だった後に判定されるので、期限後に届いた結果も採用されうる（session.cc:7919-7949）。利用者が期限内に候補を得られなかった割合は、session 側の終点で数える。
    - 上流ベンチの `predictionHitCount` は `predictionResults` が空でなければ加算されるだけで、正解との照合も、zenz 由来かの区別も無い（レビュー経由・未検証）。的中率の基準には使えない。
  - この 2 つが揃えば、案 B（フォールバック）と案 C（上位に混ぜる）を比べられる。上流の打鍵単位のベンチマーク（`Tests/.../ZenzaiTests.swift` の 440-501 行付近。レビュー経由・未検証）を参考にできる。

## 未確認のまま残ったこと
- zenz-v3.2 で読みの続きを予測したときの的中率（**確認した資料の中には**、v3.2 の読み補完の的中率は無かった）
- 予測用 prompt で、最初に `EE01` が出る頻度と、途中で切れる率（今の wire では測れない。scorer 側の計測が要る）
- `ZenzFeedbackCandidateRewriter`（単一 segment の feedback）が同じ 7 文字の閾値を使うか
- 補完値を今の validator と adoption に通したときの実際の判定（実行していない）
- 実機での遅延（中央値、p95、timeout 率）
- mozkey 同梱版の llama-server `/completion` で prompt token の logprob が取れるか
- 著者論文の学習損失の記述と、azooKey-Desktop の既定値（いずれもレビュー経由・未検証）
- 自分が読んだ azooKey の HEAD と、レビュアーが固定した `fddde10d…` が同じか
- 実際に配布物へ同梱されている重みとバイナリ。既定のファイル名は、Windows も POSIX も `zenz-v3.2-small-Q5_K_M.gguf`（main.cc:294-297、main_posix.cc:20）。lock には重みの revision と SHA-256 がある。同梱物のハッシュは照合していない

---

## レビュー反映ログ（round 1）

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| critical 1 | 「今の scorer で動く」は prompt を送れるところまでに限定 | 採用 | `empty_content`（llama_http_client.cc:931-934）、カタカナ検査の欠如、途中で切れた文字列の返却をコードで確認した |
| critical 2 | feedback の 7 文字閾値 | 採用（訂正） | proto 1039/1040 の既定、config_handler.cc:390-394、session.cc:1232-1246, 7157-7160 で、再利用に閾値が渡ることを確認した。記録には効かないことも proto のコメントと session.cc:1222-1225 で確認した |
| critical 3 | session.cc・main.cc・main_posix.cc が未添付で未検証 | 一部不採用 | 自分はこれらを直接読んでおり、各行を確認済み(自) と明記し直した。deferred の 250 と 300 の食い違いは採用して訂正した |
| should-fix 1 | 1-2 の「不可」は強すぎる | 採用 | 「学習目的から外れ、評価が無い」に弱めた。論文の記述はレビュー経由・未検証と明記した |
| should-fix 2 | 未確定ローマ字の prefix を取り違えている | 採用 | Prediction.swift の resolvePredictiveInputSource を読み、確かめた。`minLength` の意味も訂正した |
| should-fix 3 | フォールバックと cache の適用範囲 | 採用 | prefixText で絞った後に空か判定すること、cache は読みの生成だけを省くことを、コードで確認した |
| should-fix 4 | prompt の並びと、n_probs では entropy を再現できない | 採用 | 上流の builder と mozkey の builder を比べて確認した。全語彙の entropy は上位 N からは復元できない |
| should-fix 5 | validator と worker の断定に条件を補う | 採用 | zenz_live_corrector.cc:53-125 で、上書きされるのは待機中の要求だけと確認した。validator の他の拒否条件も明記した |
| should-fix 6 | 並べ替えの選択肢を狭めている | 採用 | PR #305（次の 1 文字方式）と issue #27174 を開いて確認し、案 A に加えた |
| should-fix 7 | 遅延の見積もりを撤回し、実測の指標を書く | 採用 | 見積もりを撤回し、中央値・p95・timeout 率と、的中率・空振り率を書いた |
| nit 1 | azooKey-Desktop の既定はオフ | 採用（未検証と明記） | 自分では開いていないので、レビュー経由・未検証として記載した |
| nit 2 | wire の「3 項目だけ」という表現 | 採用 | ヘッダの全 field を列挙し、「区別する field もタグの検査も無い」と言い換えた |

## レビュー反映ログ（round 2）

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| critical 1 | 「そのまま使える」は通信経路に限る。Windows も同じ | 採用 | Windows の main.cc:1190-1229 と 1309-1323 を読み、greedy と固定 stop、`empty_content` を確認した。2-1 に範囲を明記した |
| critical（feedback の再利用） | 既定設定では、短い key の fast path 再利用は起きない | 反映済み（round 1） | round 1 で訂正済み。config_handler.cc:390-394、session.cc:7157-7160 と内容が一致する |
| should-fix 1 | `FocusDisplayedPassiveSuggestion` 経路の取りこぼし | 採用 | ms-ime.tsv:60/128、session.cc:10070- と 11093-11164 で、Predict() の前に昇格の経路があることを確認した。2-3 の 4 に追記した |
| should-fix 2 | live preedit の表示と学習では、成立条件が違う | 採用 | session.cc:8214-8227（strict）、8376-8388（表示だけでは記録しない）、5786-5821（投影した区間の学習が 0 件なら全体の key/value にフォールバック）を確認した。2-2 の表を訂正した |
| should-fix 3 | 時間設定をどの区間の値かまで書く | 採用 | 各定数と getter、deferred の判定順（TakeResult より先）、Windows の起動待ちが下限であること（main.cc:1521-1558）を確認した。§3 を表に書き直した |
| nit | async response のログには latency が無い | 採用 | session.cc:7919-7930 に latency の項目は無い。main.cc:1698-1711 の scorer 側で計時していることを確認した。§3 に記載した |
| 指摘なし | 既定構成での生成上限 64 token | — | round 1 の記載（確認済み(自)）のまま |

## レビュー反映ログ（round 2 正式版）

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| should-fix 1 | `empty_content` の意味。先頭 EE01 率と打ち切り率は今の wire では測れない | 採用 | llama_http_client.cc:668-687, 931-934 と main.cc:1309-1323 で、空になる原因を区別できないこと、非空なら打ち切りでも ok になることを確認した |
| should-fix 2 | 別 worker にしても推論の待ちは解消しない | 採用 | main.cc:1077（`--parallel 1`）、1792（HandleClient を逐次処理）、posix_server.cc の ServeOne が同期であることを確認した |
| should-fix 3 | 待機窓の判定順と、timeout 率の測り方 | 採用 | session.cc:7919-7949 で、待機窓は TakeResult が空だった後に判定されることを、main.cc:1664-1671 で準備待ちの失敗が Timeout になることを確認した |
| should-fix 4 | 全読みで記録するなら、再利用の入口が要る。記録の条件も不足 | 採用 | 完全一致の lookup（store 1426-1440）、fast path が複数 segment に限られること（session.cc:7097-7106）、incognito・private・パスワード欄の条件（1203-1225）を確認した |
| should-fix 5 | B と C を比べる指標を分ける | 採用 | 空振り率を 5 つの指標に分けた。上流ベンチの predictionHitCount の扱いは、レビュー経由・未検証として記載した |
| should-fix 6 | possibleNexts の制約をどこまで再現するか | 採用 | 上流の isAllowedPrefix がこの向きで判定することを、round 1 で読んだコードで再確認した |
| should-fix 7 | Windows の既定モデルと、runtime lock の llama.cpp 版 | 採用 | main.cc:294-297 と llama_runtime.lock.json:25, 48 を確認した。固定版 README の n_probs の定義は、レビュー経由・未検証 |
| nit 1 | strict の拒否理由の例外 | 採用 | session.cc:191-229 と validator 896-911 で、正規化読みが空なら reading_unavailable になることを確認した |
| nit 2 | 条件文字列の切り方の差（末尾 25 文字と先頭 64 文字） | 採用 | 上流の builder と zenz_prompt_builder.cc の SanitizeConditionText を確認した |
| nit 3 | 版と検証者の表記 | 採用 | 表題を更新し、上流の版と「確認済み(自)」の意味を冒頭に注記した |

## レビュー反映ログ（round 3）

**注: レビューは 3 回が上限なので、この round 3 の反映分は astra（確認役）の再確認を受けていない。**

| # | 指摘 | 判定 | 一行理由 |
|---|---|---|---|
| should-fix 1 | B の「辞書予測が空」を、mozkey で判定できる条件にする | 採用 | 上流の判定対象は、round 1 で読んだコードで確認済み。外側で最大 3 件をまとめる処理（L1042-1069）は、自分では開いていないのでレビュー経由・未検証と明記した |
| should-fix 2 | 未確定ローマ字を、Tab で選ぶ時点でも照合する | 採用 | session.cc:4445-4462 と 9005-9039 で、ローマ字が未確定の間はサジェストを保持すること、昇格の経路があることを確認した |
| should-fix 3 | 先頭 EE01 率には、整形前の content が空かも必要 | 採用 | stop 語が除かれること、整形の処理（llama_http_client.cc:668-687）を確認した。llama.cpp 固定版の応答仕様は、レビュー経由・未検証 |
| should-fix 4 | 遅延と失敗率の分母を決める | 採用 | B では AI 候補の選択時点が存在しない場合がある、という指摘は論理的に正しい。分母と、読みの的中の定義を書き分けた |
| should-fix 5 | accepted を、Tab で選んだ証拠にしない。送信前の保護も引き継ぐ | 採用 | session.cc:6579-6614（比較が一致したら RecordAccepted）と 7446-7508（パスワード欄・privacy の検査、文脈の組み立て）を確認した |
| nit 1 | Windows のモデルパスの説明 | 採用 | main.cc:291-297 で、既定パスと、環境変数がデバッグビルドに限られることを確認した |
| nit 2 | 「ありが」の例と、一般の拒否理由を分ける。表示が 1 segment とは限らない | 採用 | validator と、表示処理の MarkPendingRomanTailOnPreedit（session.cc:8373）を確認した |
| nit 3 | 時間表の「上限」に条件を添える | 採用 | session.cc:7935-7949 と posix_server.cc:48, 467-470（wire の 0 を 30000ms に置き換える）を確認した |
