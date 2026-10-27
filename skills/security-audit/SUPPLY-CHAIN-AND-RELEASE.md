# サプライチェーン・リリースの脆弱性探索

#### このファイルを使う場面

依存解決、非信頼の貢献からのビルド、CI、リリース成果物、署名・昇格、プラグイン、配備済みソフトウェアの更新に使います。ソース・依存から利用者が動かす成果物までの信頼の受け渡しを扱います。ローカルローダーの内部欠陥は `MEMORY-SAFETY-AND-BINARY.md`、実行時権限は `CLOUD-AND-DEPLOYMENT.md` を使います。

大きな対象は依存解決、CI 隔離、成果物の来歴、リリース認可、更新／プラグインに分けます。

## Core discipline（基本原則：この分野の各エージェントへのプロンプトに含める）

```text
- 可変・既知脆弱な依存だけでは指摘にしない。解決へ影響する主体、利用ビルド、その後の実行・公開境界を示す。
- ソース ID、解決済み入力、ビルドワーカー、成果物 ID、テスト結果、署名・証明、昇格、更新の利用先まで完全性を追う。
- CI 設定は認可コードである。発火イベント、実行するコードの所有者、秘密・トークン、公開・変更権限を確定する。
- 成果物と同じ非信頼の場所から取得する checksum は独立した完全性を保証しない。信頼の根と失敗時の挙動を特定する。
- リポジトリ内の制御フローの破綻と限定したローカル検証は `confirmed`、観測できないブランチ保護、hosted runner、registry、署名サービス、本番昇格は `needs_validation` にする。
```

## 依存・ビルド入力の分類（subagent_type: `general`）

**Dependency source and namespace confusion**（依存元・名前空間の混同）
意図しない公開／私有名前空間、代替 registry、mirror、repository、URL を解決しないか調べます。名前、優先度、lockfile、checksum、別ビルド定義、プラットフォーム差、初回と更新の違いを確認します。

**Mutable and unbound build inputs**（可変・結び付けのないビルド入力）
branch、tag、未検証 submodule、取得ツール、生成資源、remote include、可変 CI Action・コンテナ tag がレビューなしに変わらないか調べます。低信頼の書き手と信頼成果物への経路が必要です。再現可能性だけでは真正性を証明しません。

**Generated-source and codegen provenance gaps**（生成ソース・codegen の来歴不足）
スキーマ、同梱アーカイブ、生成クライアント、翻訳、文書例、バイナリがソース同等のレビュー・完全性検査なしに実行・同梱内容を生まないか確認します。再生成とコミット済み出力を比較し、入力と生成器の管理者を調べます。

**Build-context inclusion**（ビルドコンテキストへの混入）
広すぎる対象・除外規則で、秘密、設定、リポジトリ情報、テスト、開発生成物が package／image に入らないか確認します。実際の資格、私的データ、特権設定の露出を確認します。

## CI・自動化の分類（subagent_type: `general`）

**Untrusted code in a privileged workflow**（特権ワークフローでの非信頼コード）
PR、Issue コメント、fork、依存更新、外部イベントが保護された秘密、書込トークン、配備権限、信頼 runner で投稿者コードを動かさないか調べます。trigger、checkout ref、承認、環境保護、権限制限を比較し、ソース外のホスト既定値を仮定しません。

**Workflow command and expression confusion**（コマンド・式の混同）
branch 名、commit メッセージ、Issue 項目、成果物名、matrix、生成出力が正規化検査なしにシェル、テンプレート式、パス、特権入力へ入らないか確認します。

**Cache, artifact, and workspace trust mixing**（キャッシュ・成果物・作業領域の信頼混在）
低信頼ジョブが書いた cache、artifact、workspace、output を高信頼ジョブが復元して実行・公開しないか確認します。キー、名前空間、作成者 ID、digest、保持、可変名での再解決を調べます。

**Automation identity overreach**（自動化 ID の権限超過）
必要な操作、repo、環境、期間以上の権限を、非信頼入力から選ぶ対象へ使えないか確認します。最小権限の欠如だけなら強化提案で、到達可能な特権操作が必要です。

## リリース・更新の分類（subagent_type: `general`）

**Build-to-promotion substitution**（ビルドから昇格までの入れ替え）
テスト、レビュー、署名、公開が同じ不変 digest ではなく可変 tag、名前、channel、ID を参照しないか確認します。copy、再梱包、CPU 統合、来歴の全工程を追います。

**Release authorization and signing-policy gaps**（リリース認可・署名ポリシーの不足）
違う workflow、repo、branch、環境、鍵役割、閾値の署名・release を受理しないか調べます。証明内の ID 主張を利用先が検証するか確認します。有効な署名だけでは不十分です。ローテーション、期限、失効はポリシー上必要なら拒否側に動く必要があります。

**Update metadata and rollback confusion**（更新メタデータ・巻き戻しの混同）
バイトだけ認証して版、製品、平台、channel、宛先、期限、rollback 状態を結び付けない、または別取引の metadata と payload を組み合わせないか確認します。原子的な設置と回復を調べます。ポリシーとの結び付けのない署名 API 呼出しだけでは不完全です。

**Plugin and extension trust expansion**（プラグイン・拡張の信頼拡大）
申告範囲を超えるホスト権限、他の発行者への成り済まし、真正性・機能検査前の install/update hook を調べます。意図的に同じユーザーの任意プラグインを入れること自体は特権境界ではありません。

## Universal moves（共通の調べ方）

- 公開 digest・導入更新から逆に、ソース、生成入力、資格、worker、cache、テスト、認可をすべて追います。
- 非信頼イベントと保護イベントを比較し、永続チャネルの越境に不変 ID と作成者の信頼性を要求します。
- 失効鍵、取得失敗、証明欠落、平台の部分リリース、rollback、registry 障害を調べます。失敗時ポリシーも完全性の一部です。

## Validation rules（検証ルール：報告前にすべて適用）

1. 低信頼主体、操作可能な source/cache/artifact/metadata、信頼ジョブ・updater、未許可公開・混入・漏えい・特権実行を示します。
2. 壊れた受け渡しの成果物 ID を証明します。異なる可変名・未固定 digest が実際の利用先へ届く必要があります。
3. 固定版の package manager、repo host、registry、署名の既定値を確認します。不明な hosted 制御は `needs_validation` です。
4. 無害な fixture repo、ダミー資格マーカー、ローカル registry／設定、本番以外の名前空間で限定検証します。本物の release を公開・変更しません。
5. 完全なソース上の受け渡しと意味ある結果だけを `confirmed` にします。`needs_validation` は所有者が観測すべき branch、runner、registry、署名、配備の事実を示します。
