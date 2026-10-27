# クライアント側・ブラウザーの脆弱性探索

#### このファイルを使う場面

SPA、ブラウザー拡張、組み込み WebView、Service Worker、オフラインアプリなど、ブラウザーで重要な信頼判断や非信頼内容の表示を行う場合に使います。攻撃者が影響できる DOM、ウィンドウ間メッセージ、ストレージを確認します。URL フラグメント、`window.name`、`postMessage`、過去のキャッシュなど、サーバーに届かない入力も含みます。

`ATTACK-CLASSES.md` と併用します。本書はブラウザーの入力元・処理先、origin 境界、永続化、サイト間の状態推測を扱います。WebView ブリッジのネイティブ側は `DESKTOP-MOBILE-AND-LOCAL-IPC.md`、サーバーの CSRF・セッション・認証コールバックは `WEB-PROTOCOL-AND-AUTH.md` を使います。

## Core discipline（基本原則：この分野の各エージェントへのプロンプトに含める）

```text
- 操作可能な入力元と、実行または漏えいを起こす処理先を特定し、攻撃者のデータが届くことを示す。
- 被害者のセッション、別 origin、共有の永続状態への影響が必要。自分だけへの注入や自分のデータの開示は指摘にしない。
- フレームワークのエスケープ、同一 origin ポリシー、CSP、COOP/CORP、Service Worker の範囲、現在の noopener の既定動作を実際の防御として確認する。
- ストレージとキャッシュは origin 単位で共有され、ログイン後も残り得る。書き手、読み手、アカウント・テナント・ワーカーのどの遷移で消すかを特定する。
- 完全なソース証拠と限定的なローカルブラウザーテストがある場合だけ `confirmed` にする。レンダラー、拡張権限、デプロイ済みヘッダー、ブラウザーポリシーが不明なら `needs_validation` にする。
```

## DOM・オブジェクト状態の攻撃分類（subagent_type: `general`）

**DOM-based XSS**（DOM ベース XSS）
`location`、`document.referrer`、`window.name`、メッセージ、ストレージ、ブラウザー内の文書状態から、`innerHTML`、`outerHTML`、`document.write`、文字列評価 API、実行可能 URL、jQuery HTML API、フレームワークのエスケープ回避機能へ追います。フレームワークがエスケープする埋め込みは指摘にしません。

**DOM clobbering**（DOM clobbering（DOM の名前衝突））
攻撃者が注入した `id` や `name` が、後で信頼されるグローバル変数、フォーム属性、設定、初期化フラグを隠さないか調べます。属性が保持されるマークアップ経路と、上書きされた値のセキュリティ上重要な利用の両方が必要です。

**Prototype pollution and gadget chain**（プロトタイプ汚染とガジェットの連鎖）
攻撃者のキーが deep merge やパス代入などの再帰書き込みでプロトタイプを変更し、その値を使う到達可能な処理が認可、実行、移動、表示を変えるか確認します。`JSON.parse`、浅いコピー、利用先のない汚染だけでは不十分です。

## origin 間通信・ネットワークの攻撃分類（subagent_type: `general`）

**`postMessage` origin and source trust**（`postMessage` の origin・送信元の信頼）
厳密な origin 許可リストや、同一 origin の複数フレームでは期待する `event.source` を確認せず、`event.data` で機密操作をしないか確認します。送信先 `*` による想定外の埋め込み側への漏えいも調べます。部分一致、前方・後方一致、境界なしの正規表現は origin 検査になりません。

**Cross-site WebSocket request use**（サイトをまたぐ WebSocket の利用）
`Origin` 検査やチャネル専用トークンなしに、信頼できない origin から被害者の Cookie を伴う接続を受け入れ、データを読み書きできないか確認します。接続アップグレードと重要なメッセージハンドラーを両方検証します。

**Credentialed CORS trust**（認証情報を伴う CORS の信頼）
`Origin` を反射・弱く照合し、認証情報付きで機密応答を返さないか確認します。単なるワイルドカードと認証情報の併用はブラウザーが拒否するため、実際に許可・反射される origin とデータ漏えい・変更だけを報告します。

## Service Worker・ストレージの攻撃分類（subagent_type: `general`）

**Service-worker registration and scope takeover**（Service Worker の登録・範囲の乗っ取り）
攻撃者の内容がワーカースクリプト、過剰な `Service-Worker-Allowed` の対象パス、完全性検証のない更新用 import にならないか確認します。最終 URL、応答 MIME、origin、範囲、全 import の管理者を確認します。意図した範囲の同一 origin ワーカーは欠陥ではありません。

**Service-worker cache and identity confusion**（Service Worker のキャッシュ・ID 混同）
アカウント、テナント、認可状態、要求モードを考慮せず個別応答を保存し、ログアウトや切り替え後に返さないか確認します。fetch の振り分け、キャッシュ名・キー、ナビゲーション代替、削除処理、エラー・オフライン時の他人の応答を調べます。

**Browser-storage disclosure and stale authorization**（ストレージの漏えい・古い認可）
`localStorage`、`sessionStorage`、IndexedDB、Cache Storage、拡張ストレージ、クライアント状態にトークン、私的応答、下書き、認可判断が残り、別アカウントや低信頼の同一 origin 部品に読まれないか確認します。トークンの保存だけでは不十分で、現実的な低権限の読み手、または失効・ログアウト後の継続利用が必要です。

**Cross-context storage and broadcast confusion**（コンテキスト間のストレージ・配信の混同）
`storage` イベント、`BroadcastChannel`、Shared Worker、origin 全体のキャッシュが現在のセッションに結び付かない ID・コマンドをタブ間で運ばないか確認します。アカウント、非公開／通常ウィンドウ、テナントの切り替え、古いタブによる新認証状態の上書きを調べます。

## サイト間の情報漏えい分類（subagent_type: `general`）

**XS-Leaks and cross-origin state oracles**（XS-Leaks・origin 間の状態推測）
被害者の認証情報が送られるとき、読込／エラー、フレーム・ウィンドウ状態、転送、時間、キャッシュ、応答サイズから保護状態を区別できないか調べます。私的オブジェクト、役割、アカウントの存在など、機密を表す具体的な条件が必要です。一般的な時間のばらつきや公開リソースの可用性は指摘にしません。

**Window and opener state disclosure**（ウィンドウ・opener の状態開示）
別 origin ウィンドウの許可されたメタデータや移動結果が保護状態を漏らすか、残った opener／名前付きウィンドウが特権的な移動へ影響するかを確認します。COOP、フレーム保護、`noopener`、厳密な origin、状態の機密性を確認します。

## UI 偽装・移動の攻撃分類（subagent_type: `general`）

**Clickjacking**（クリックジャッキング）
状態変更を伴う画面で `frame-ancestors`、`X-Frame-Options` 相当の隔離が機能するか調べます。機密操作がフレーム内で完了できることが必要です。読み取り専用内容のヘッダー欠落は強化提案です。

**Client-side navigation confusion**（クライアント側の移動先の混同）
スキーム・宛先ポリシーなしに入力が転送・移動を制御し、`javascript:` や `data:` へ進めないか確認します。逆タブナビングは `window.opener` を明示的に残す、隔離なしの `window.open`、暗黙の `noopener` がないブラウザーに限って検討します。

## Universal moves（共通の調べ方）

- DOM、移動、ワーカー、メッセージ、ストレージの処理先から逆に追い、ブラウザー専用入力とサーバー側入力を区別します。経路を止めるべきブラウザーポリシーを記録します。
- ローカルのテスト origin とダミーアカウントで、切り替え、ログアウト、ワーカー更新、オフライン、古いタブを確認します。本番の利用者・origin・共有サービスは使いません。
- XS-Leaks はソースとローカルブラウザーで証明した条件だけを列挙し、推測を防ぐ応答ヘッダーや表示方式を特定します。

## Validation rules（検証ルール：報告前にすべて適用）

1. 入力元、処理先、ブラウザーポリシー、影響 origin／セッション、観測できる変更・漏えいを引用します。
2. プロトタイプ汚染は再帰書き込みと重要な利用処理、DOM clobbering は残るマークアップと上書き値の使用を証明します。
3. ワーカー・ストレージでは攻撃者の書き込みやキャッシュが別アカウント、テナント、後の認可状態に届くことを証明します。
4. 通信、CORS、WebSocket、XS-Leaks は厳密な origin／送信元検査と露出する保護状態・操作を示します。CSP、COOP/CORP、Cookie、SameSite で既に止まらないか確認します。
5. 完全なクライアント経路と限定的なローカル証拠があるものだけ `confirmed` にします。`needs_validation` には所有者が確認すべきヘッダー、拡張権限、ブラウザーバージョン、レンダラー挙動を具体的に書きます。
