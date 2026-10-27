# デスクトップ・モバイル・ローカル IPC の脆弱性探索

#### このファイルを使う場面

デスクトップ／モバイルアプリ、特権ヘルパー、更新処理、ローカルデーモン、WebView ホスト、ディープリンク、ブラウザーの native messaging、IPC クライアント／サーバーに使います。攻撃者は、ダウンロード文書、遠隔 Web 内容、別アプリ、別 OS ユーザー、サンドボックス内プロセス、低権限アカウントなどです。すべてのローカル利用者を同一視せず、開始時点の能力を明示します。

WebView のブラウザー側は `CLIENT-SIDE.md`、メモリとローダーは `MEMORY-SAFETY-AND-BINARY.md`、更新の真正性は `SUPPLY-CHAIN-AND-RELEASE.md` を使います。

## Core discipline（基本原則：この分野の各エージェントへのプロンプトに含める）

```text
- 別アプリ、別 OS ユーザー、隔離された子、非信頼文書、遠隔 origin など、現実的な攻撃者を定める。同じアカウント・権限内での自己影響は越境ではない。
- パス、プロセス名、bundle/package ID、自称送信者は相手認証ではない。OS の相手資格、コード ID、capability ハンドル、保護されたチャネル状態を使う。
- ブリッジやヘルパーは構文解析後の操作と最終リソースを認可する必要がある。信頼する UI・仲介があっても、攻撃者が影響できる引数は非信頼である。
- OS サンドボックス、署名、entitlement、権限、キーチェーン ACL、公開コンポーネントのポリシー、確認画面の動作は、版が固定され見えるなら実際の制御として扱う。
- ソース証拠と限定したローカル／エミュレーター検証には `confirmed` を使う。署名、マニフェスト統合、OS 版、端末ポリシー、インストーラー ACL、パッケージ化が不明なら `needs_validation` にする。
```

## ディープリンク・コールバック・移動の分類（subagent_type: `general`）

**Custom-scheme and deep-link ambiguity**（カスタムスキーム・ディープリンクの曖昧さ）
現在のセッションと一度限りのコールバックに結び付かない状態変更、取込、認証完了、アカウント選択を別アプリやページから呼べないか調べます。URI 正規化、重複クエリ、scheme/host/path、公開ハンドラー、古い・再送リンクを確認します。

**App and account handoff confusion**（アプリ・アカウント受け渡しの混同）
OAuth、SSO、マジックリンク、招待、端末ペアリング、パスワードレス、支払いの戻り先が別アプリ、プロファイル、テナント、処理中取引にならないか確認します。開始アプリ ID、セッション、アカウント、プロバイダー、操作、期限に state を固定します。

**File-open and intent authority confusion**（ファイルを開く操作・Intent の権限混同）
関連ファイル、共有 Intent、ドラッグ＆ドロップ、クリップボード、通知操作、open-file イベントで、型、必要な送信者信頼、現在の意図、最終対象を確認せず特権処理が動かないか調べます。

## WebView・ネイティブブリッジの分類（subagent_type: `general`）

**Navigation-origin to bridge confusion**（移動先 origin とブリッジの混同）
同梱内容専用の JavaScript／native ブリッジへ遠隔・攻撃者フレームが届かないか確認します。呼出時と毎回の移動、転送、子フレーム、ポップアップ、エラー／代替ページで origin を検査します。URL 前方一致や初回だけの検査では不十分です。

**Over-broad native bridge capabilities**（過大なブリッジ機能）
Web 内容が汎用ブリッジから任意のファイル、コマンド、IPC、秘密、システム操作を選べないか調べます。メソッド許可、正規化引数、ユーザー／テナント権限、操作・確認要件、返り値の漏えいを確認します。

**Webview file and universal access**（WebView のファイル・universal access）
ファイルアクセス、universal access、混在内容、デバッグ、独自プロトコルが origin を結合し、遠隔内容からローカルファイル・特権スキーム・内部 origin を読めないか調べます。読める保護内容がない設定不足は強化提案です。

## ローカル IPC・公開コンポーネントの分類（subagent_type: `general`）

**IPC peer-authentication gaps**（IPC 相手認証の不足）
Unix ソケット、名前付きパイプ、XPC、Binder、D-Bus、native messaging、RPC、共有メモリ、ループバック接続で、OS 資格、コード ID、サンドボックストークン、チャネル所有権を確認するか調べます。背後に重要な操作・開示があることが必要です。

**Claimed principal versus channel identity**（自称主体とチャネル ID の不一致）
認証されたプロセス・チャネルと異なるユーザー、テナント、プロファイル、機能を引数から選べないか確認します。各メソッドと資源を自称 ID ではなく相手の資格へ固定します。

**Exported service, activity, receiver, or provider overreach**（公開 service・activity・receiver・provider の権限超過）
外部から呼べるモバイル部品・自動化口がアプリ専用操作を行わないか確認します。最終統合マニフェスト、Intent フィルター、権限／署名レベル、パス許可、別名を調べます。パッケージ後の状態不明は `needs_validation` にします。

**IPC lifecycle and correlation confusion**（IPC の寿命・要求対応の混同）
予測可能 ID、ハンドル再利用、古いチャネル、継承 descriptor、誰でも書けるソケット、再起動で、別の相手の操作への応答・取消・再利用が可能か調べます。ソケット、ロック、ポート、共有マッピングの作成権限と削除を確認します。

## 特権ヘルパー・ファイルの分類（subagent_type: `general`）

**Privileged helper as confused deputy**（ヘルパーを使う confused deputy）
低権限の呼出元が操作ごとの認可なしに特権コマンド、ファイル、サービス、ユーザー、設定を選べないか確認します。sudo/polkit/UAC/XPC の規則と、ヘルパー自身による正規化引数の検証を調べます。

**Install, update, and repair path trust**（インストール・更新・修復パスの信頼）
認可後に低信頼主体が変更できるマニフェスト、スクリプト、パッケージ、リンク、作業ディレクトリ、修復状態を特権処理が読まないか確認します。承認は不変な内容と安全な宛先に固定します。

**Local file ownership and TOCTOU**（ローカルファイルの所有権・TOCTOU）
検査後の入替、シンボリックリンク、マウント、大文字小文字・正規化の変更を特権読書きが追わないか調べます。descriptor 相対操作と最終所有権の検証を確認します。開いた後の解析は `MEMORY-SAFETY-AND-BINARY.md` を使います。

**Credential-store and local-secret boundary mismatch**（認証情報ストア・ローカル秘密の境界不一致）
キーチェーン／キーストア、トークン、バックアップ、ログ、クリップボード、通知プレビュー、設定を別アプリ・プロファイル・ユーザーが読めないか確認します。意図した同じ OS アカウントだけが読める平文は自動的に脆弱性にはなりません。低信頼の読み手と資格の権限を示します。

## アプリ状態・端末ライフサイクルの分類（subagent_type: `general`）

**Account switch, logout, and device restore leakage**（切替・ログアウト・端末復元での漏えい）
キャッシュ、バックグラウンド処理、ウィジェット、通知、ローカル DB、WebView 保存、生体認証の承認が切替後も残り、次のアカウントに現れないか調べます。バックアップ／復元と複数プロファイルを確認します。

**Pending-action and user-presence confusion**（保留操作・本人の立会いの混同）
通知、ウィジェット、ショートカット、共有画面、生体認証、遅延処理が表示と違う操作を認可したり、期限後・別プロファイルで実行したりしないか調べます。確認を正規化した操作、資源、アカウント、現在の前面状態に固定します。

## Universal moves（共通の調べ方）

- 全プロセス、部品、ローカル口、URI スキーム、ファイル関連付け、WebView origin、ヘルパーを列挙し、OS ID、実行権限、呼出元、操作を記録します。
- 統合マニフェスト、entitlement、インストール規則、native messaging 登録、プロトコルハンドラー、ACL 作成など、最終パッケージ入力を読みます。元宣言は後段で上書きされ得ます。
- 隔離したマシン／エミュレーターとダミープロファイル・非機密フィクスチャで確認します。他人のアプリ、秘密、本番サービスに触れません。

## Validation rules（検証ルール：報告前にすべて適用）

1. 攻撃者の初期能力、越える OS／アプリ主体、入口、受理される引数・状態、未許可操作・開示を示します。
2. OS 隔離、相手資格、署名、entitlement、権限、同意、インストール制御を確認します。未知のパッケージ・実行時事実は `needs_validation` にします。
3. WebView は移動／origin 制御と native の特権処理、IPC は相手認証と資源認可、ヘルパーは最終正規化宛先を示します。
4. ローカル検証を制限し、ダミー内容・アカウントを使います。越境結果で止め、永続化や広範なシステム変更に進みません。
5. ソースとローカル証拠が揃うものだけ `confirmed` にします。`needs_validation` は必要な OS、マニフェスト、署名、ACL、端末状態の事実を明示します。
