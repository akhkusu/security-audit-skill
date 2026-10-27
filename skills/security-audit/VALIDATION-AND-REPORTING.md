# 候補検証・構造化出力・独立検証・報告

### 第3段階：すべての候補を独立して検証する

網羅性レビューが問題なしとなるか、早期停止を明記した後、第2段階の候補と同一ソースの過去確認済み記録を安定 fingerprint・根本原因で統合します。重複のない confirmed・needs_validation の提案を、それを探索していない新しい `general` 検証担当へ渡します。過去 confirmed も、探索除外されていても現在の検証を通します。探索・過去成果物は読めますが、現ソースの全引用箇所を読み直し、安全に再現できる決定的な検査は独立実施します。

各検証担当に正規小文字の一意 ID と `<output-dir>/agents/<verifier-id>/scratch/`、親所有 artifacts を与えます。書けるのは scratch だけです。渡すのは候補、関連単位の検査・成果物パス、経路解釈に必要な architecture、該当補足の正確な検証ブロック、下の昇格手順、ソース／ローカル実行境界、report-schema の3判定分岐全文、同 fingerprint の過去記録だけです。他検証担当の結論を渡してはいけません。

#### 候補検証担当へのプロンプト

```text
この候補を書いたのはあなたではありません。ソースと範囲を制限したローカル証拠から反証してください。
配備先や外部・共有サービスへ接続しません。対象コードは承認された OS 強制 sandbox 内だけで
実行します。外部通信なし、空から許可変数だけの環境、対象・tool 読取専用、scratch のみ書込、
低い資源・実時間上限が必要です。1つでも不足なら実行せず、正確な不足能力を needs_validation の
阻害要因にします。終了後も scratch はすべて対象制御と考えます。sandbox と全 process 終了後、
信頼する親側コードだけが、このプロンプトにそのまま含めた昇格手順で事前指定ファイルを保存します。
担当と対象コードは保持成果物を書きません。昇格が不能・失敗ならそのファイルを証拠にしません。

1. trace・evidence の全 file、正の line、scope、description を検証します。最初が本当の低信頼入口、
   最後が主張する処理先・越境効果か確認します。
2. 経路上の最も強い検証、ID、認可、正規化、ライフサイクル、framework、封じ込め制御を再構成します。
   類似製品が architecture にあれば、同型かを調整材料として記し、却下理由には使いません。
3. confirmed 提案は可能なら最小観測結果を独立再現します。入力、interface の形、条件、影響する
   ダミー主体・資源を確認し、強い結果を推測せず、最小結果後は継続しません。
4. likelihood、impact、confidence、ソース修正案が証拠だけに対応するか確認します。
5. needs_validation は阻害要因が本当にソース・ローカル観測外か判断します。ソースで反証できれば
   rejected。不明点が依然決定的なら needs_validation のまま、local・所有者観測計画を正確で
   非破壊的にします。
6. 同じソース由来根本原因は全状態で fingerprint を維持します。

前後に文章を付けず JSON オブジェクトを厳密に1つ返してください。
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
record は渡された schema の decision に対応する verdict 分岐へ厳密に一致させます。
修正した記録が探索担当の文章を置き換えます。
```

次の昇格手順を全候補検証プロンプトへそのままコピーします。

```text
成果物の昇格手順（信頼する親側コードだけが実施）：
担当者は参照するだけです。実施するのは親であり、担当者ではありません。

実行前に親が、信頼できる継承不可の scratch/・artifacts/ root directory descriptor を
開いて保持し、許可する scratch 相対ファイルと個別・合計バイト上限を記録します。
descriptor は担当や sandbox へ渡しません。sandbox と全プロセスの終了後に、
信頼する親側コードが許可ファイルを1つずつ昇格します。

1. 宣言相対パスを検査し、絶対、空、`.`、`..`、symlink の成分を拒否する。
2. 保持した scratch-root descriptor から各親を no-follow の directory-relative 操作で
   走査する。パスで開き直さない。
3. 末端を no-follow・非 blocking で開く。
4. `fstat` で通常ファイル、link count が厳密に1、個別・合計上限以内と確認する。
5. descriptor から読む間も上限を再度強制する。
6. 検証サイズだけコピーし、再 `fstat` で ID、型、link count、サイズの変化を拒否する。
7. 宛先も保持した artifacts-root descriptor から全親を no-follow で走査する。
   既存成分は実 directory とし、不足分は排他的に作成して no-follow で開き直し検証する。
8. 末端は link を追わず排他的に作成し、通常ファイルかつ link count 1 を検証する。
   どちらのパスも開き直さず、検証した元 descriptor からコピーする。
9. 非 POSIX では同等の競合安全な API を使う。
10. scratch の再帰コピー・glob、artifacts 内への archive 展開、symlink、FIFO、socket、
    device、directory、hard link、変化中・上限超過ファイルの開封・昇格は禁止。
11. 検査がない・強制不能・失敗なら scratch 項目を捨てる。決定的証拠なら正確な昇格阻害要因と
    共に `needs_validation` を維持する。
```

needs_validation を confirmed に上げられるのは、完全経路と限定観測結果を独立に確立した場合だけです。特定の配備・実行事実が不明なら confirmed 提案を needs_validation へ下げます。ソース、ローカル挙動、可視制御、意味ある影響の欠落、不可能な前提で反証されたら rejected にします。推測を保管するために needs_validation を使いません。

親は同じ fingerprint が返るか確認します。本当に別の根本原因なら例外です。訂正を統合し、全関連単位へ判定を記録し、fingerprint あたり最終記録を1つにします。不正形式・文章で包んだ結果は補修せず破棄し、予算があれば新担当で再実行します。不足なら未完了規則に従い未検証 candidate に留めます。

検証証拠で台帳を更新する場合、検査の agent_id は検証担当の正規 ID とし、空でない相対 reviewed_paths を記します。単位のパスは全検査の和集合です。source は artifact null、local は親が正常昇格した `agents/<check.agent_id>/artifacts/` のファイルだけです。単位の元割当所有者は変えず、独立所有の探索・検証証拠を共存させます。過去記録を引き継ぐ planned 単位に所有者がなければ、再確認担当が所有者となり、その最初の検査で fingerprint 付き candidate へ移します。

厳密な総呼出し予算で全候補を賄えなければ、SKILL の決定的な予算規則に従い未完了にします。未検証候補は台帳だけに残し、どの判定でも findings に入れません。

### 第4段階：`findings.json` を出力・検証する

親は独立に判定済みの全記録を fingerprint 順に `<output-dir>/findings.json` へ書きます。

- `confirmed`：完全なローカル実行証拠、条件、具体的修正、可能性・影響・総合深刻度、確信度を持つソース根拠のある脆弱性。
- `needs_validation`：正確な未解決阻害と、該当するローカルまたは所有者観測計画を最低1つ持つ候補。
- `rejected`：検証で反証された候補。証拠が変わらないのに将来同じ主張を繰り返さないため保持します。

出力直前に report-schema を読みます。`additionalProperties: false` のため、探索結果ラッパーの項目を持ち込みません。判定契約を区別します。

- confirmed は root_cause、intended_behavior、conditions、execution、remediation、severity、confidence を使います。claimed_root_cause、blockers、validation_plan、reason は禁止です。execution は対象に合う API/HTTP、CLI、library、message、file fixture、browser、生成 policy、local harness を使い、稼働先に依存させません。observed_result は空でない事実です。
- needs_validation は claimed_root_cause、trace、evidence、blockers と、空でない validation_plan.local または deployment を最低1つ使います。両方入れるのは別々の不明点を解決する場合だけです。severity、execution、remediation、reason、確認済み用 root_cause は禁止です。
- rejected は claimed_root_cause、trace、evidence、reason を使います。severity、execution、remediation、blockers、validation_plan、確認済み用 root_cause は禁止です。

全記録に安定 fingerprint、title、description、相対ソースパスを付けます。複数 trace は entrypoint から sink まで、中間だけ propagation。1要素なら entrypoint または sink です。総合深刻度は実証影響を越えません。

```sh
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

構造・意味エラーをすべて直してから続行します。findings 検証器は 5 MiB、トップレベル1,000指摘、入れ子64段を越える入力を拒否し、エラー表示は100件までです。合格が証明するのは形式と台帳の整合だけです。

### 第5段階：新しい担当で最終記録を確認する

最終 confirmed・needs_validation の各記録に、新しい `research` 担当を1人ずつ並列起動します。探索メモではなく構造化記録を検証し、ソース／ローカル境界を守ります。

quick では第3・5段階をまとめ、同じ第3段階担当が以下も確認して最終 schema 記録を返します。候補につき新しい独立担当は2人でなく1人です。他 profile は2回を分けます。confirmed の独立レビューは全 profile で必須です。

confirmed の確認項目：

1. trace/evidence の全相対パス、行、scope、操作説明。
2. 実在する入口 interface と正確なローカル入力形式。
3. 全条件、parser/policy、可視の防止層、観測ローカル結果。
4. 影響主体・資源と実証影響。
5. 現実的 likelihood、実証 impact、それを越えない overall の区別。
6. remediation と code_changes が信頼の移動だけでなく不変条件を強制するか。

needs_validation の確認項目：

1. ソース経路が実在し、記載した claimed_root_cause だけを裏付ける。
2. 全阻害要因が決定的で、既にローカルで答えられるものではない。
3. 一般的懸念でなく境界と具体的に起こり得る結果がある。
4. 正確な計画欄が最低1つある。local は限定 fixture、deployment は所有者の設定・ID・経路・policy・runtime 観測。適用外の計画を作らず、配備先へ監査通信を送らない。
5. 同じ根本原因の過去・現記録と fingerprint が一致する。

各担当は前後の文章なしで `{"decision":"verified","fingerprint":"..."}` または `{"decision":"replace","reason":"...","record":{...}}` を厳密に1つ返します。置換記録は対応する3判定の schema に従います。不正形式・文章付き結果は第3段階同様、補修せず破棄し、予算があれば新担当で再実行します。

より強い判定への昇格（confirmed への昇格を含む）、または根本原因、trace、実行入力・観測結果、実証影響、深刻度の重要な変更は、そのまま最終適用しません。探索、第3段階、置換提案のいずれにも参加していない新しい独立担当へ置換全体を渡します。現ソースを再確認し、決定的ローカル結果を実行境界内で独立再現して verified または別置換を返します。重要置換はその新検証後だけ適用し、さらに重要置換が出れば新担当で繰り返します。予算・独立性を確保できなければ disputed record を findings から除き、台帳は未解決 candidate に保ち、正確な incomplete_reason で未完了にします。意味・証拠を変えない表現・行位置修正は直接適用できます。

置換適用後は毎回両検証器を再実行し、関連台帳判定を更新します。別根本原因は新 fingerprint を与え、独立した候補検証を通してから追加します。全台帳 candidate に独立最終判定があり、保持記録すべてが第5段階を通る場合だけ complete にします。

confirmed だけの検証は禁止です。誤った needs_validation は所有者の時間を浪費し、偽の前提を残します。

### 第6段階：最終記録から対象環境に依存しない報告を作る

findings に残す全記録が第5段階を通った後だけ、記録、台帳、管理項目に保持した hardening から文章を作ります。未完了でも独立検証済み記録は報告できますが、未解決の台帳 candidate をすべて特定し、指摘として扱いません。文章で判定、深刻度、阻害、実証影響を変えません。

#### `REPORT.md`

次を記載します。

1. profile、scope、予算と使用／予定回数、source ref、sandbox 内のソース・ローカル限定の実行方針、過去結果の利用、延期・範囲外を明記。同一ソース confirmed の引継ぎと変更ソース再検証も示します。quick、部分、予算制限、未完了は部分監査と明記します。検証予算切れなら未完了と全未検証 fingerprint・単位を列挙し、指摘とは呼びません。必須レビューが未実行ならどれか示し、網羅完了を主張しません。
2. 短いセキュリティ状態の要約。
3. 確認済み表：深刻度、題名、影響境界、1行の観測結果。
4. 各 confirmed：相対ソース位置、低信頼主体、対象固有の限定再現、条件、実結果、影響、優先度根拠、最小修正。
5. 独立した **追加検証が必要（NEEDS VALIDATION）** 表：題名、相対経路、正確な阻害、限定ローカルの次手順、安全な所有者観測。深刻度を付けず確認済み脆弱性と呼びません。
6. 強化提案と、ソース上の良い実装を別記。
7. 台帳由来の covered、candidate、blocked、deferred 件数、重要な除外、最終網羅性レビュー結果。

rejected を指摘にしません。過去の相違や範囲判断の説明に必要な場合だけ fingerprint に言及します。

#### `FINDINGS-DETAIL.md`

medium・high・critical の各 confirmed について、完全なソース経路と対象固有のローカル再現を転記します。

- 順序付き相対 trace・evidence
- ダミー攻撃者・主体と影響ダミー資源
- native 入力・呼出し・fixture と正確で限定した手順
- 観測出力と証明する安全上の不変条件
- 条件と封じ込め
- ソース修正と回帰ケース

#### `NEEDS-VALIDATION.md`

未解決記録ごとに、ソース経路、検証済み証拠、正確な阻害、影響境界、該当する限定ローカル・所有者観測計画を転記します。深刻度なしの優先付けした候補とし、稼働環境への試験案内に変えたり、不明な配備事実を仮定したりしません。

HTTP はインターフェースの一例で、既定ではありません。library は関数、parser は fixture、CLI は command、desktop は IPC・file、infrastructure はローカル生成 policy を使えます。対象にない endpoint、外部 account、稼働環境を要求しません。

報告の大きさは証拠に合わせます。confirmed 0件でも構いません。残る調査範囲・検証の限界とその結果を示し、LOW の指摘を作り出しません。
