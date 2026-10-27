# 事前調査

### 第1段階：ソースの全体像を把握し、調査範囲を計画する

親エージェントは `run-metadata.json` を初期化し、`SKILL.md` の調査前の厳密な予算判定を行います。その後、探索の前に各担当の一時領域と共有台帳を作成します。予算判定を満たさなければ未完了を記録し、事前調査担当を起動しません。読むのは対象とローカルのビルド・設定状態だけです。配備先、外部 IdP、registry、broker、cloud API、共有サービスへ接続しません。

複数の `research` 担当を並列起動します。構造化した事実を親へ返し、ファイルは書きません。

**担当 1a：製品・技術構成・ローカル動作**

```text
<target> を読み、通信せず次を返してください。
1. 製品の種類、利用者、運用者、通常の信頼に関わる操作。
2. 言語、framework、build system、runtime、ローカルで見える配備モデル。
3. リポジトリ相対の入口とサブシステム境界。
4. 手元の依存だけでオフライン実行できる正確な build/test コマンド、書込先、処理する対象制御入力。事前調査中は実行しないでください。
5. ローカル文書・依存から根拠を示せる類似ソフトウェア・プロトコル。なければその旨。
6. 限定実行を妨げる不足 toolchain・runtime 情報。
リポジトリ相対の file:line 参照を付け、ソース上の事実だけを返してください。
```

**担当 1b：主体・権限・制御**

```text
ID、認可、隔離、特権を確立する全ソースを読み、次を整理してください。
1. 各低信頼主体と、設計上許される操作。
2. 各入口の認証・相手 ID。
3. 資源ごとの認可と tenant/owner 範囲。
4. process、browser、workload、CI、plugin、model/tool、device、local IPC の権限。
5. 権限変更、確認、失効、回復、代替経路。
6. ソースで見える制御と、未観測の配備事実に依存する制御。
信頼境界と制御箇所をリポジトリ相対 file:line 付きで返してください。稼働先の到達性を推測しないでください。
```

**担当 1c：入口・複製・処理先**

```text
外部・低信頼入力が入るソース上の全箇所を列挙してください。
- HTTP/browser、RPC/message/protocol、file/archive/document、CLI/env/config、plugin/dependency/CI、cloud event/IAM selector、model context/tool argument、mobile/deep-link/webview、local IPC。
各入口から主な変換、保存・派生コピー、セキュリティ上重要な処理先を追い、見える制限と同じ効果への並列経路を記録してください。
相対パスと行番号を返してください。漏れなく調べますが、実行や入力送信はしないでください。
```

**担当 1d：ローカル実行・配備の可視性**

```text
test、build 定義、manifest、packaging、保守対象の環境別上書きを読み、次を返してください。
1. 必須 OS サンドボックス内のダミーデータで境界を検証できる、小さいオフライン test・既存 fixture。
2. 外部・共有依存なしの隔離 loopback namespace で動かせる process。
3. 依存取得、成果物公開、有料・provider API、共有状態変更を行うコマンド。今回は禁止と明記。
4. ソースで確定できず、決定的なら needs_validation が必要な配備制御と適用先。
5. リポジトリが決定的に選ぶ場合だけ、各配備モードの最終有効ソース経路。
6. ローカル平台で、空から許可変数だけを入れた環境、外部通信禁止、対象・toolchain の読取専用 mount、scratch のみ書込、CPU・memory・process・file size・disk・wall time 制限を強制できるか。不足は対象コード実行の阻害要因。
7. 信頼する親側コードが、事前指定 scratch ファイルを、パス内に閉じた no-follow descriptor 走査、非 blocking の通常ファイル確認、宛先全親の no-follow 走査、排他的な通常ファイル作成、個別・合計サイズ制限で保存できるか。不足は scratch を証拠として使う際の阻害要因。
```

4担当で把握できない重要な別配備モード・サブシステムには追加の事前調査を割り当てます。予算で不可能なら省略を隠さず、その領域を `deferred`、理由 `budget_cannot_reserve_critics_and_validation` の単位にし、報告書へ不足を記します。

## 過去の監査結果の取り込み

作業選択前に、同じ repo の利用可能な全 `coverage-ledger.json` と `findings.json` を読みます。

- 各記録・単位のソース位置、制御、条件、ソース由来 ID を現在と比較します。
- 同じソース・条件・適格な証拠が有効な過去の `confirmed` だけを、同じ fingerprint で今回の候補へ引き継ぎます。`prior_status: "prior_confirmed_same_source"` の今回の `planned` 単位へ結び、同じ根本原因だけ探索除外にします。第3段階の再検証担当が割当所有者となり、その最初の再確認で単位を fingerprint 付き `candidate` にします。
- 関連ソース・条件が変われば `prior_confirmed_changed_source` の再検証単位を計画します。探索除外せず、過去判定を前提にしません。
- 過去の `needs_validation`、`deferred`、`blocked`、`out_of_scope`、変更済みソースを今回の作業にします。優先度の参考にはなりますが重複除外・抑制のキーにはなりません。
- 未解決の過去 `needs_validation` は現ソースで経路を確認した後だけ同じ fingerprint で引き継ぎます。`prior_status: "prior_needs_validation"` の `planned` 単位へ結び、阻害要因を保持します。第3段階の担当が所有者・最初の検査を担い、fingerprint 付き `candidate` へ移し、最終検証にも含めます。
- 過去の棄却は、失敗した経路や欠けた条件を変える新証拠がない限り古い主張です。同じ棄却で除外するのはその主張だけで、調査単位全体ではありません。
- 台帳の欠落・非互換を記録し、調査範囲が空だったとは扱いません。

利用したパスと source ref を `run-metadata.json` に記録し、`architecture.md` には調査範囲への影響だけをまとめます。

## アーキテクチャ要約・補足ガイドの選択

親は `<output-dir>/architecture.md` を約1,000語以内にまとめます。

1. 製品、主体、通常権限、保護資源。
2. 担当 1a が根拠を示した類似製品がある場合、その製品が許容する安全上のトレードオフ。工数・深刻度の調整に使い、実証した指摘を却下する理由にはしません。実際に問題となった同型欠陥があれば裏付けを強めます。有用な比較対象がなければ省略します。
3. 技術構成、見える配備経路、オフライン build/test 制約。
4. 入口と主要な入力→処理先・ライフサイクル経路。
5. 各信頼境界とソースで見える最も強い制御。
6. 調査開始用のリポジトリ相対パス。
7. 過去の不足、変更・阻害された再検証、同一ソースの確認済み除外。
8. [ATTACK-CLASSES.md](ATTACK-CLASSES.md) に基づく選択ガイドと、それが必要なソース上の境界。

担当ごとの通常分類、補足ブロック、理由付き除外は `architecture.md` ではなく各台帳単位に記録します。要約の長さを保ち、プロンプトへの割当を機械検査可能にするためです。

言語・依存名があるだけで補足ガイドを選びません。「このファイルを使う場面」に該当する信頼境界を見つけたことが理由になります。関連分類を別担当が見るからといって、可視の境界を除外しません。

## 再現可能な調査台帳

`<output-dir>/coverage-ledger.json` はトップレベルが JSON 配列です。入口 × 信頼境界 × サブシステム × 通常／補足分類の重要な組ごとに単位を作ります。粒度は profile に従い、`quick` は範囲内全体を1サブシステムに、`deep` はライフサイクルも分けます。部分監査では範囲内を割り当て、見つけた範囲外も `out_of_scope` で残して将来の作業に使います。

各軸には表示ラベルと、`canonical_refs` の安定したソース由来値を付けます。表示が変わっても同じ対象には同じ参照を使います。入口相対パス＋公開スコープ、ソース定義の route/message、境界を定義する制御、package パス、正確な攻撃分類ブロックなどです。ブロック参照は `FILE.md#` と、太字または見出しの正確な分類名をつなぎます。HTML アンカーではありません。補足の共通節は括弧注記より前の見出し（例：`Core discipline`）を使います。表示名の小文字化・slug 化で作らないでください。日本語版でも参照用の英語名を保持します。

情報を落とす slug を使わず `coverage_id` を生成します。

1. 各参照は正しい Unicode scalar の NFC とし、可視文字を含め、前後空白、制御・format・行／段落区切り・default-ignorable 文字を禁止します。
2. UTF-8 バイトを RFC 3986 の percent encoding にし、`A-Z a-z 0-9 - . _ ~` だけを残し、それ以外を大文字 `%HH` にします。
3. 符号化した `surface`、`boundary`、`subsystem`、`attack_class` を `::` で連結し、あれば `lifecycle` も追加します。

quick の subsystem は固定値 `profile/quick/all-in-scope-subsystems` です。参照・ID に wave、agent、verdict、severity、行番号を含めません。割当前に `coverage_id` の辞書順で整列し、重複 ID は必ずエラーにします。同じ ID で意味が違えば正規 ID 衝突であり、統合・黙った上書きは禁止です。同じ意味の組を異なる正規参照で表す場合も検証器が拒否します。

各単位の形式：

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/router.ts#POST /users/:id",
    "boundary": "src/authz.ts#requireOwner",
    "subsystem": "packages/api",
    "attack_class": "ATTACK-CLASSES.md#Access control"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["repo/relative/path"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Access control",
  "selected_companion_blocks": ["FILE.md#section"],
  "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
  "prior_status": "new|prior_confirmed_same_source|prior_confirmed_changed_source|prior_needs_validation|prior_deferred|prior_blocked|prior_out_of_scope|prior_covered_same_source|prior_covered_changed_source|prior_rejected_claim_changed|none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

ライフサイクルが重要なら `canonical_refs.lifecycle` と表示用 `lifecycle` を両方加えます。通常分類がない場合だけ `ordinary_attack_class_block` を null にします。補足リストには適用分類と各ガイドの `Core discipline`、`Universal moves`、`Validation rules` を含めます。検討した未選択ブロックはすべて、除外を裏付けるソース事実と共に `excluded_blocks` へ記録します。

管理項目は追加できますが、上の意味を持つ項目は安定させます。互換台帳があるとき初出の入口は `prior_status: "new"`、互換台帳なしなら `none` です。過去の延期・阻害・範囲外・変更済み単位が今は範囲内なら `planned` にします。同一ソースの調査済みも今回の台帳に残し、変更箇所、重要な状態遷移、明確な矛盾を先に割り当て、再確認の要否を網羅性レビューで判断します。

`attempts` は網羅性レビューが再開した証拠付き割当の追記専用履歴です。再割当前に、元の `wave`、`status`、`agent_id`、`reviewed_paths`、`local_checks`、`result_fingerprints`、`unresolved` と、根拠付き `reassignment_reason` をそのまま保存します。履歴にできる状態は `blocked`、`covered`、`candidate` のみです。現行と同じ状態・証拠条件を保ち、wave は現在未満で厳密に増加し、所有者・成果物を保持します。次の割当は wave を増やし、新しい所有者と空の現行証拠で開始します。profile・予算で再割当できなければ wave を増やして、所有者 null、証拠空、停止理由付きの現行 `deferred` にします。過去所有者の証拠を現行へコピーせず、新しい最終状態は新証拠だけを含め、履歴は変更しません。

次の状態表を厳守します。

| 状態 | 単位の `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | null | 空 | 空 | 空 |
| `not_applicable`, `out_of_scope`, `deferred` | null | 空 | 空 | 空でない理由 |
| `in_progress` | 正規 ID の所有者 | 空 | 空 | 空 |
| `blocked` | 正規 ID の所有者 | 両方に所有者付き部分証拠 | 空 | 空でない阻害要因 |
| `covered` | 正規 ID の所有者 | 両方が空でない | 空 | 空 |
| `candidate` | 正規 ID の所有者 | 両方が空でない | 空でない | 任意 |

agent ID は `^[a-z0-9][a-z0-9_-]{0,63}$` に一致し、Windows device 名は禁止です。小文字必須で、大文字小文字だけ違う別名を認めません。単位の `agent_id` は割当所有者です。各検査も固有の `agent_id` と空でない `reviewed_paths` を持ち、単位のパスはその厳密な和集合にします。source-only は `artifact: null`。local は信頼する親側が `agents/<check.agent_id>/artifacts/` 直下の部分木へ昇格した通常ファイルが必要です。これにより探索・検証担当の証拠が共存できます。scratch、出力根のファイル、link、特殊ファイル、別検査所有者の成果物は証拠ではありません。

台帳が調査範囲の主張です。要約、担当数、一般的な「認証を確認済み」という文は証拠になりません。第2段階で単位を閉じる根拠は、担当の構造化結果のパスと検査だけです。

初期作成後、親の全更新後、第6段階の前に次を実行します。

```sh
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

上限は 5 MiB、入れ子64段、10,000単位、入れ子コレクション各1,000要素、走査値500,000件です。エラー表示は100件までです。実用的な単位では 5 MiB に約2,000〜5,000件しか入らず、件数より先に容量上限へ届きます。割当・調査範囲の主張の前に全エラーを直します。
