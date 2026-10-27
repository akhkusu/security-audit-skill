# 脆弱性の探索

### 第2段階：調査台帳に基づく探索を繰り返す

親は台帳の `planned` 単位を `general` 担当へ割り当てます。無関係な境界を一緒にせず、各単位を調べられるだけの担当を用意します。同じサブシステムの密接な単位は1人で持てます。人数上限で黙って未割当にせず、予算外の単位は `deferred`、理由 `budget_cannot_reserve_critics_and_validation` と明示します。

予算・profile が人数を制限する場合は、優先順と根拠を台帳に記録します。(1) 認証済みより未認証・低信頼の入口、(2) 資格、他テナントデータ、コード実行、リリース権限など価値の高い資源の境界、(3) 同一ソース再調査より過去の不足、再検証、変更ソース、(4) 推測的な分類より対象種別で確認済みの成果を出してきた分類、の順です。同順位は `coverage_id` の辞書順で再現可能にします。

起動前に割当を `in_progress` にし、小文字の正規 `agent_id` を設定し、担当の scratch と親所有 artifacts を作ります。担当はソースと親の文脈を読み、自分の scratch だけに書き、Task tool で構造化結果を1つ返します。保持成果物、対象ソース、architecture、台帳、findings、他担当のファイルを変更しません。

## 必須の探索プロンプト

次をこの順序で含めます。

1. 2文の役割説明：割当単位でソース根拠のある安全上の不変条件の破綻を探すこと、末尾の契約に合う JSON オブジェクトを厳密に1つ返すこと。
2. `architecture.md` 全文をそのまま。
3. 割当 coverage ID、subsystem、boundary、相対開始パス、各単位の台帳の割当ブロック対応表。
4. 選択ブロックを正確にそのまま：通常分類は ATTACK-CLASSES の各ブロック、補足は `Core discipline`、各選択分類小節、`Universal moves`、`Validation rules`。通常ブロックは自己完結しており補足型の共通節はありません。名前だけを送ってはいけません。
5. 除外した通常・補足ブロックと個別理由。
6. 下記の基本探索方法、続けて成果物の昇格手順。
7. 下記の基本検証ルール。
8. 同一ソースの引継ぎ済み確認済み除外（fingerprint、title、root cause だけ）と、重複禁止の他担当 coverage ID。
9. 一意な scratch/artifact パス、安全な ID、事前指定の昇格許可リスト・バイト上限、構造化結果契約。下の結果ブロックと、report-schema の confirmed・needs_validation 分岐を全文そのまま含めます。

同じ経路が複数分野を越える場合は複数補足を選べます。制約をまとめて渡します。範囲は調査義務であり、除外作業の重複許可ではありません。想定外の別境界は `uncovered` に返し、親が安定単位を作って次 wave に割り当てます。

#### 基本探索方法：全探索プロンプトに含める

```text
## 防御目的の脆弱性探索方法

目標は、ソースに根拠のある安全上の不変条件の破綻と最小修正を見つけることです。
境界結果を越えて被害を広げません。ソース調査と限定したローカル実行に留めます。
配備先、provider API、registry、ID システム、broker、共有サービス、他人へ接続しません。
ローカルのダミーデータだけを使います。

深くコードを読んでください。各入力を解析、ID、認可、正規化、状態、派生コピー、最終処理先まで
追います。同じ効果の兄弟、旧版、batch、retry、取消、移行、error 経路も読みます。
制御は存在だけでなく同等性を、コンポーネント間では保証と次段の前提の一致を比較します。

具体的な不変条件から始めます。
1. 低信頼主体と初期能力を示す。
2. 受理される値、操作、状態遷移、資源選択を示す。
3. 拒否・結び付け・隔離・制限・失効すべき制御を特定する。
4. 判断後の正確なソース経路を追う。
5. 最小の影響ダミー記録、誤返り値、プロセス完全性への効果、ローカル共有資源への効果で止める。
6. 不変条件を強制するソース変更と回帰テストを示す。

深さの制限：割当境界へ届く経路、またはその境界が依存する保証の経路だけを追います。
安全・不安全どちらでも条件が確定したら、その調査を止め、covered・candidate・blocked または
uncovered として構造化結果に記録し、探し続けません。

異常系と不一致を調べます。インターフェースが受理する範囲で、欠落、空、0、負数、最大、超過、
重複、混合符号化、古い、失効、順序違反、並行、移行途中、依存失敗、巻戻しを確認します。
各 parser・policy の受け渡しで正規化と単位を比較します。複数段階では各出力を次の前提とし、
後段境界を仮定しません。未確定の前提は阻害要因として記録します。

high・critical 候補に再利用可能な根本原因があれば、自分の coverage ID 範囲で字句・構造・論理の
類似経路を探します。同じ原因は統合しますが、各変種の条件・影響は独立に確立します。
他担当の単位を調べません。現在の単位がない変種は uncovered に返します。

主張を解決する最小のローカル検査を使います。対象制御の build、test、process、browser、emulator、
fuzzer、fixture は親承認の OS 強制 sandbox 内だけで動かせます。外部通信禁止、空から許可変数
だけの環境、対象・tool の読取専用、scratch のみ書込、低い CPU・memory・process・file size・disk・
wall time 制限が必須です。隔離 loopback は local fixture にだけ許可します。
不足があれば実行せず、その正確な阻害要因で needs_validation を返します。既存 unit test、
最小関数 harness、ダミーテナント呼出し、小さな不正 fixture、決定的競合、生成 policy を優先し、
tool を導入・取得しません。

正確な入力、コマンド、制限、最小結果を記録します。環境は再現に必要な許可変数名と安全な非秘密値
だけを記録し、ホスト全環境、継承変数、資格、認証状態、無関係なパスを取得しません。
対象プロセスは scratch だけに書きます。sandbox と全 process 終了後、信頼する親側コードだけが
このプロンプトにそのまま含めた昇格手順で事前指定ファイルを保存できます。
担当と対象コードは保持成果物を書きません。決定的証拠の昇格が不能・失敗なら正確な阻害要因で
needs_validation を返します。可用性への負荷、稼働先、本物の資格、公開、最小効果後の継続は禁止です。

ソース外の配備、browser、provider、broker、OS、proxy、package、secret、ID の事実は、有無の証明に
なりません。それが決定的なら、必要な観測と安全な所有者確認を明記して needs_validation にします。
```

#### 成果物の昇格手順：全探索プロンプトへそのままコピーする

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

#### 基本検証ルール：全探索プロンプトに含める

```text
## 候補の判定基準

1. 候補には、最も強い可視制御を含め、主張する根本原因の証拠と完全な相対ソース経路が必要。
2. confirmed の提案には、制限されたローカル観測結果、明示した境界を越える意味ある影響、
   完全な条件が必要で、見える防止層が存在してはいけない。
3. crash を code execution、通常作業を共有可用性、同一主体の操作を権限獲得へ格上げしない。
4. 必須事実がソース・ローカルで見えなければ needs_validation。正確な阻害要因を示し、
   深刻度や推測の完結した結果を与えない。
5. 影響主体・資源のない best practice 不足は除外または強化提案であり指摘ではない。
   ソースで反証された候補は needs_validation ではない。
6. 同じ根本原因は全状態で同じソース由来 fingerprint を使う。
   `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` に一致し、行、wave、agent、severity、verdict を含めない。
7. 基準を満たす候補がなければ空の candidate 配列を返す。
```

## ローカル検証の境界

実行は確認のためであり、影響拡大のためではありません。

- **必須 OS sandbox 内だけで許可**：既存依存の offline build、ダミー状態の隔離 loopback、unit/integration test、小 fixture、sanitizer、限定 fuzz・回帰 test、決定的競合、ダミー browser/emulator、ダミー ID の manifest 生成・policy 評価、外部・有料呼出しの mock。
- **禁止**：稼働・配備先への通信、今回の隔離検査用に起動していないサービス、ネット経由の依存導入、本物の account・資格、本番 data、共有 queue・cloud・runner・registry・署名・release、公開、stress・飽和・費用生成、最小ダミー境界結果後の継続。

sandbox は空の環境から開始し、対象へ外部通信・ホスト書込先を与えず、重そうな検査だけでなく全検査に低い資源・時間上限を設けます。終了後も scratch は対象制御です。SKILL の no-follow、パス制限、通常ファイル、サイズ制限の親側手順でだけ昇格します。sandbox・昇格能力の不足で根拠のある候補を消さず、正確な阻害を needs_validation に記します。

## 探索結果の構造

前後に文章を付けず、JSON オブジェクトを厳密に1つ返します。

```json
{
  "units": [
    {
      "coverage_id": "割当 ID を1つ指定",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["repo/relative/path"],
      "checks": [
        {
          "agent_id": "この検査の正規所有者 ID",
          "reviewed_paths": ["repo/relative/path"],
          "invariant": "この単位で確認した具体的な制御",
          "method": "source|local",
          "result": "ソースまたは限定検査で確立したこと",
          "artifact": "local は agents/<agent-id>/artifacts/file、source は null"
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["指摘ではない具体的な強化提案"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "reason": "独立した再現可能な調査単位が必要な理由"
    }
  ]
}
```

`candidates` の各項目は schema に従いますが、`verdict` の代わりに `proposed_verdict` を使います。

- `proposed_verdict: "confirmed"`：schema の confirmed 分岐の必須項目から verdict だけを除き、fingerprint、title、description、root_cause、intended_behavior、順序付き trace、evidence、conditions、対象固有の execution、remediation、severity、confidence を含めます。execution は実施済みの限定ローカル検査だけを説明します。payloads は最小入力・fixture・native invocation、observed_result は実際の出力です。総合深刻度は観測影響を越えません。
- `proposed_verdict: "needs_validation"`：同分岐の verdict 以外の必須項目、fingerprint、title、description、claimed_root_cause、順序付き trace、evidence、空でない blockers、該当する local または deployment を最低1つ持つ validation_plan を含めます。適用外の計画を作りません。severity、execution、remediation、reason、confirmed 用 root_cause は含めません。deployment は所有者の観測であり、稼働先探査の依頼ではありません。

割当 ID は units に厳密に1回ずつ出します。covered は所有者、空でないパス・検査、未解決なし・候補なし。candidate は同じ証拠条件を満たし、fingerprint を持てる唯一の状態。blocked は所有者付き部分調査でパス・検査・未解決があり fingerprint はありません。ソースパスはすべて repo 相対で、絶対・traversal を禁止します。複数 trace は entrypoint で始まり sink で終わり、中間は propagation とします。各検査に正規小文字 agent_id とパスを持たせ、単位のパスはその和集合にします。source は artifact null、local は親が正常昇格した `agents/<check.agent_id>/artifacts/` の通常ファイル1つです。検証者が独立所有の証拠を追加しても探索者の所有権を奪いません。scratch、出力 root、他検査所有者の成果物は参照しません。

## 親による統合・台帳更新

結果を検査し、各単位を割当 ID 1つへ対応させ、その単位だけ更新します。重複・欠落 ID、危険な担当 ID、成果物付き source、親が検査所有者の artifacts 部分木へ昇格していない local を拒否します。reviewed_paths、checks（台帳では local_checks）、成果物、fingerprint、未解決を写します。hardening は第6段階に使うため、意味を持つ標準項目とは別の関連単位の管理項目に保持します。失敗・不正・裏付け不足の結論は再割当用 planned に戻します。未着手の予算・profile 外単位は、証拠空・所有者なし・理由付き deferred とし、部分証拠を隠しません。更新後に台帳を検証し、不正なら次の割当を行いません。1人がある単位を閉じ、別単位は候補・阻害として返せる契約です。

候補は fingerprint、次に根本原因で統合します。同じ原因の複数入口・効果は、最も強い完全経路を持つ1候補にします。関連しても独立した制御欠落は別 fingerprint です。重複は関連単位に記録し、検証へ重ねて送りません。

## 網羅性レビューの反復

各探索 wave の直後、予約した1回で新しい `research` 担当を起動します。architecture、割当ブロック表を含む全台帳、現在の候補 fingerprint・状態、過去台帳の不足要約を渡します。ソースを読むだけで、書込・対象実行はしません。出力は厳密に次の JSON です。

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "ソースに裏付けられた調査範囲の不足"
    }
  ],
  "reassign_ids": ["閉じられなかった既存 ID"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

未整理の入口、未検査の並列経路、状態遷移の欠落、単位のない選択分類、不当な除外、証拠なしの完了、対応単位のない過去 needs_validation・変更箇所を確認します。提案は調査範囲であり指摘ではありません。stop は担当自身の評価で、missing_units と reassign_ids がともにない場合だけ true。継続は stop 単独でなく以下の親の条件で決めます。resolved_prior_leads は親が関連単位・過去候補を解決済みにし、担当のソース根拠付き理由を記録します。

範囲外・ソース／ローカル境界外の提案は拒否し、受理分の正規 ID を生成して重複排除します。同一ソースの完了単位は証拠を提供できますが、過去 deferred・blocked・out_of_scope・変更済み単位は現在の作業であり提案を抑制しません。ID 衝突は統合せず失敗にします。正当な reassign_id が blocked・covered・candidate の証拠を持つなら、その最終状態をそのまま attempts へ、ソース根拠付き reassignment_reason と共に追加します。所有者、検査、成果物、fingerprint、未解決を保持し、現 wave を増やします。次は新所有者が証拠空の in_progress から始め、新しい最終結果だけを現行に書きます。履歴の証拠をコピーしません。次割当前に ID 整列と台帳検証を行います。

standard/deep で、wave 後レビューの受理不足・正当な再割当がなく planned もなければ、別に予約した新しい担当で最終問題なしレビューを行います。そこにも受理作業がない場合だけ網羅完了にします。発見があれば queue へ戻し、探索→直後レビュー→最終レビューを反復します。時間・資源で停止するなら未着手をすべて deferred にし、理由を保持して報告します。黙った wave・人数上限を完了の証拠にしません。

quick は探索1回と最終レビュー1回だけです。受理 missing_unit は理由 `quick_profile_final_critic` の deferred とします。証拠付き再割当は現最終状態を attempts に保存して wave を増やし、現行を証拠空・未割当・同理由の deferred にします。2回目の探索・レビューは禁止です。部分監査の範囲外発見はレビューで報告し、親が理由付き out_of_scope にします。quick は事前宣言した早期停止であり、網羅完了の証拠ではありません。

予算も同様に制限します。各 wave 前に探索数、検証予約、直後レビュー、保持する最終レビュー（quick は1回の最終兼直後のみ）を残予算と比較し、優先順で探索を縮めます。必須予約が収まらなければ探索せず、planned を理由 `budget_cannot_reserve_critics_and_validation` の deferred にします。レビュー提案も同じ優先 queue に入り、予算を拡張しません。候補が検証予約を越えたら SKILL の未完了規則に従って探索を止め、fingerprint 順で検証し、未検証は未解決 candidate として残し、指摘にしません。
