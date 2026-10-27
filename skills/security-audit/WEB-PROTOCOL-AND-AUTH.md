# HTTP プロトコル・認証の脆弱性探索

#### このファイルを使う場面

HTTP の解析、キャッシュ、ブラウザー認証、ID の境界を持つ Web アプリ、API、プロキシ、CDN、ゲートウェイ、独自サーバーに使います。セッション、JWT、OAuth/OIDC、SAML、パスワード回復、MFA、パスキー、API キー、mTLS も対象です。`ATTACK-CLASSES.md` のアクセス制御が「主体に操作権限があるか」を問うのに対し、本書は「どの主体・要求・認証強度・トークンの操作か」を通信・認証層が混同しないか調べます。

第1段階から分類を選び、要求境界／キャッシュ、ブラウザー認証、ID 連携、強認証／回復、サービス資格、セッション寿命に分けます。詳細不明の managed proxy 背後の単独サーバーでは、ソースから確認できる smuggling の範囲は小さく、proxy・独自 parser では大きくなります。

## Core discipline（基本原則：この分野の各エージェントへのプロンプトに含める）

```text
- 境界解析・キャッシュの指摘には、同じ要求・応答・キーの2つの解釈が必要。両コンポーネントと各側の正規化値を示す。
- 資格ごとに署名・秘密の検証と、必要な issuer、audience、origin、RP、client、session、主体、資源、認証強度、期限、一度限りの状態との結び付けを探す。
- Host、Forwarded、X-Forwarded-*、Origin、Referer、転送先、callback state、要求由来 URL は信頼判断である。影響 ID・応答まで追う。
- ヘッダー、Cookie 属性、MFA 確認、レート制限の欠落だけでは不十分。不正要求の受理、他主体への影響、認証強度低下、秘密開示が必要。
- 完全なソース証拠と限定したローカル要求・token テストだけを `confirmed` にする。必要な proxy、IdP、browser、certificate、secret、配備設定が見えなければ `needs_validation` にする。
```

## HTTP 境界・キャッシュの分類（subagent_type: `general`）

**Request framing and desynchronization**（要求境界・同期ずれ）
前段と後段で長さやヘッダー正規化が異ならないか調べます。複数の `Content-Length`、`Transfer-Encoding`、HTTP/2・3 の変換、名前の正規化、禁止接続ヘッダー、CR/LF 変換を確認します。片側が現在の要求に、もう片側が次の要求に割り当てる正確なバイトを示します。

**Web cache poisoning through unkeyed input**（キーにない入力によるキャッシュ汚染）
キーにない要求値が保存内容・重要ヘッダーを変えないか確認します。転送 host/scheme、Cookie、query 正規化、言語／端末、認可状態を含む全応答差分とキーを比較します。

**Cache deception and private-response caching**（キャッシュ欺瞞・私的応答の保存）
私的動的パスを公開静的資源と解釈するか、ID・認可入力がポリシーから抜けないか調べます。edge の保存判断とアプリの経路解析、suffix／path parameter 正規化、応答指示を比較します。

**Host and forwarded-header trust**（Host・転送ヘッダーの信頼）
非信頼の host／proxy 情報が絶対 URL、tenant 配送、callback、reset link、cache key、認可用 client IP を決めないか調べます。誰が設定でき、入口で利用者指定値を除去するか確認します。

**Response-header injection**（応答ヘッダー注入）
非信頼データが危険な制御文字・正規化で `Location`、`Set-Cookie`、CSP などへ入らないか確認します。フレームワークの拒否を確認し、重要な応答変化を必要とします。

## ブラウザーセッションの分類（subagent_type: `general`）

**Ordinary CSRF**（通常の CSRF）
ブラウザーが自動送信する資格で、実効的な CSRF token、same-site 結び付け、厳密な Origin/Referer 検査なしにサイト間の状態変更を受理しないか調べます。form、JSON 類似、multipart、method override、互換経路を網羅します。SameSite は実際の Cookie・browser 条件で評価し、login CSRF と cross-site subresource の違いも確認します。

**Session fixation and invalidation**（セッション固定・無効化）
login、アカウント切替、MFA 完了、代理 login、権限変更で ID が更新されるか、logout、password 変更、失効、無効化後に使えないか確認します。サーバー、refresh token、署名 Cookie、WebSocket、cache、代替 endpoint を調べます。

**Cookie scope and transport**（Cookie 範囲・通信）
過大な `Domain`・`Path`、安全でない通信、別部品が異なって選ぶ兄弟 Cookie との競合を調べます。現実的な低信頼 origin、ネットワーク位置、browser 経路から資格を取得・交換できない属性欠落は強化提案に留めます。

## ID 連携の分類（subagent_type: `general`）

まず役割を確定します。redirect 許可・code 発行など認可サーバーの制御を、利用側クライアントへ要求しません。検証・結び付けの欠陥は成果物を受け取るコンポーネントの責務です。

**JWT verification and claim binding**（JWT の検証・claim の結び付け）
署名、サーバー固定 algorithm・鍵元、`exp`、`nbf`、`aud`、`iss` を確認します。`kid`、`jku`、`x5u` は非信頼の鍵選択値として、重複・header 正規化、未検証 decode を調べます。信頼 issuer の署名でも、別サービス用 token はここでは無効です。

**OAuth/OIDC request and callback binding**（OAuth/OIDC の要求・callback の結び付け）
対象が認可サーバーなら厳密な `redirect_uri` 所有権、session-bound `state`、必要な PKCE・code、ID token の issuer/audience/signature/nonce、複数 IdP の選択先を確認します。初回、再試行、mobile/deep-link、連携経路を比較します。

**SAML signed-object and assertion binding**（SAML の署名対象・assertion の結び付け）
検証した要素と ID に使う要素が同じか確認します。未署名・代替経路、安全な XML parser、正規化差、有効窓、audience/recipient、要求対応、再送状態を調べます。

## MFA・パスキー・アカウント遷移の分類（subagent_type: `general`）

**MFA enrollment and assurance downgrade**（MFA 登録・認証強度低下）
登録、交換、無効化、回復 code、信頼端末、代替 login に必要な事前認証強度を確認します。第1要素だけで、必要な再認証なしに第2要素を登録・交換できないか、古い・無効な要素が session を認可しないか確認します。

**Step-up binding and bypass**（追加認証の結び付け・回避）
成功した challenge が別 session、account、tenant、action、API 要求を格上げしないか、別経路が検査を省略しないか調べます。主体、現在 session、目標強度、必要な操作・資源、期限、一度限りの完了に固定します。UI、API、batch、回復、再開を比較します。

**WebAuthn and passkey verification**（WebAuthn・パスキーの検証）
登録では challenge、RP ID、期待 origin、資格、user/userHandle、algorithm、必要な本人検証を開始 session に結び付けます。認証では challenge、RP/origin、資格の所属、署名、必要な立会い／本人検証を確認します。アカウント発見・連携の userHandle と資格対応を調べます。signature counter は後退を clone の兆候に使う製品でのみ意味があります。

**Account linking and identity collision**（アカウント連携・ID 衝突）
IdP、passkey、email、phone、device、外部 account の追加には現在の認証済み session、新 ID 所有権の検証、必要な追加認証、開始 account に固定した callback state が必要です。解除・再連携・招待受諾で verified ID や tenant が衝突しないか確認します。

**Password reset and broader recovery**（パスワード再設定・その他の回復）
回復 token、support/admin 回復、backup code、端末移行、email/phone 変更は弱点になり得ます。乱数、user/action 結び付け、期限、一度限りの状態、rate／計数、配送 URL、旧 token・session 無効化を確認します。公開された account の存在差だけでは自動的に指摘にしません。

## API キー・mTLS の分類（subagent_type: `general`）

**API-key scope and resource binding**（API キーの範囲・資源の結び付け）
サーバー記録より広い tenant、resource、action、environment で通るか、引数で固定範囲を上書きできないか調べます。検索、prefix／完全 secret 検証、公開用と秘密鍵の型混同、scope、更新、失効 cache、一括 API を確認します。

**API-key exposure and unsafe transport**（API キーの露出・危険な通信）
bundle、URL、redirect、log、error、build、応答から低信頼主体へ出ないか調べます。「key」と呼ぶ公開 ID は秘密ではありません。種別と漏えいで得る権限を確認します。

**mTLS peer and application-identity confusion**（mTLS 相手・アプリ ID の混同）
任意 peer の証明書 ID header を信用する、chain は検証しても攻撃者 subject を誤 account へ対応する、別 trust domain・usage・audience・validity を受理しないか調べます。proxy 終端なら、その proxy だけ接続でき、入力 ID header を除去し、backend が無害化済み ID を要求へ固定するか確認します。

**Certificate lifecycle fallback**（証明書寿命のフォールバック）
期限、失効、欠落、更新失敗で bearer のみ・匿名へ黙って切り替わるか、長寿命 pooled connection が失効後も認可を保持しないか調べます。配備先失効情報不明は `needs_validation`、リポジトリ内の fail-open 分岐はソースから確認可能です。

## Universal moves（共通の調べ方）

- 全資格・challenge を発行 → 保存 → 送信 → 利用 → 更新 → 失効まで追い、通常、error、retry、migration、legacy、account switch を比較します。
- 同じ ID への入口と同じ機密操作への全経路を列挙します。実効ポリシーは最も弱い並列経路で決まります。
- parser、proxy、router、cache、app の正規化を横並びで比較します。稼働先に送らず、同じ限定要求 fixture を各部品へ渡します。
- 回復・連携の前後の account 図を描き、現主体、新 ID の証明、必要強度、callback/session 結び付け、失効効果を各接続に記します。

## Validation rules（検証ルール：報告前にすべて適用）

1. ソース可視性を確認します。proxy chain、edge key、IdP、証明書 trust、browser Cookie、secret、配備認証は外部かもしれません。未知の基盤動作を欠落と断定せず、正確な `needs_validation` にします。
2. 境界・cache は両部品と parse/key 差を示し、限定 fixture で要求間・ユーザー間・私的応答の影響を確認します。
3. token、MFA、passkey、連携、回復、API key、mTLS は検証行と主体／session／resource／origin／audience／action／強度の欠落を引用し、不正遷移・資格を server が受理することを示します。
4. CSRF は自動送信資格、変更経路、受理される cross-site 形式、Cookie policy、有効検査の欠落を示します。読取専用や自動送信されない bearer 必須経路は対象外です。
5. framework・library の既定値を確認します。版・設定不明なら `needs_validation` にし、未確認の critical を低深刻度の確認済みに変えません。
6. 完全なソース経路と未許可 ID・状態・開示の観測がある場合だけ `confirmed` にします。`needs_validation` は不明点と安全なローカル検証・所有者観測を示します。
