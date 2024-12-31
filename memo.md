2024.12.28

# OAuth 2.0 認可コードグラント

## 概要

認可コードグラントは、サーバーサイド Web アプリケーションで使用される最も一般的で安全な認可フローです。以下に具体的な手順を示します。

## 認可フローの手順

### 1. 認可リクエスト

クライアントアプリケーションがユーザーを Google の認証画面にリダイレクトします。

#### リクエスト URL

```http
GET https://accounts.google.com/o/oauth2/auth
    ?response_type=code
    &client_id=xxx.apps.googleusercontent.com
    &state=abcd
    &scope=https://www.googleapis.com/auth/calendar.events
    &redirect_uri=http://localhost/callback
```

#### パラメータ一覧

| パラメータ    | 説明                                         | 必須 |
| ------------- | -------------------------------------------- | ---- |
| response_type | 認可コードグラントフローを示す（`code`固定） | ○    |
| client_id     | アプリケーションの識別子                     | ○    |
| state         | CSRF 攻撃を防ぐためのランダムな文字列        | ○    |
| scope         | 要求する権限の範囲                           | ○    |
| redirect_uri  | 認可後のコールバック URL                     | ○    |

### 2. 認可レスポンス

ユーザーが認証・認可を完了すると、指定された redirect_uri にリダイレクトされます。

#### レスポンス URL

```http
GET http://localhost/callback
    ?state=abcd
    &code=4/0AanRRrvi_umBD4TsbKWrI9u4sWX1fpd3b5ScGS8WNFIGW4g7ESGHbFu2-oyOysmC98y-FA
    &scope=https://www.googleapis.com/auth/calendar.events
```

#### 検証項目

- state パラメータが元のリクエストと一致することを確認
- 認可コードは一時的なもので、速やかにアクセストークンと交換する必要がある

### 3. アクセストークン取得リクエスト

認可コードをアクセストークンと交換します。このリクエストはサーバーサイドで実行する必要があります。

#### リクエスト仕様

```http
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

code=4/0AanRRrvi_umBD4TsbKWrI9u4sWX1fpd3b5ScGS8WNFIGW4g7ESGHbFu2-oyOysmC98y-FA
&client_id=xxx.apps.googleusercontent.com
&client_secret=
&grant_type=authorization_code
&redirect_uri=http://localhost/callback
```

#### セキュリティ要件

- `client_secret`は機密情報として扱い、公開しない
- POST メソッドを使用する
- HTTPS で通信する

### 4. アクセストークンレスポンス

認可サーバーからアクセストークンが返されます。

#### レスポンス例

```json
{
  "access_token": "ya29.a0ARW5m75ZAP2xtxUVzXlS7vFcWZzBeVg_xvNCS0bBdFZ-z1emAMUjxKrcXr6Ly2mjKpwpFfZ2x1j8fPUvWzupBKvOc5UMIM37weOHDNwrkQUIizTBlDI2Owdj254MCQI93meddH21PzSoL-7rNPpvyLRFpfBlMT7aWs6bQithaCgYKAcMSARMSFQHGX2Mi-fBl3zfiMH-Ws5FMa3rONA0175",
  "expires_in": 3599,
  "scope": "https://www.googleapis.com/auth/calendar.events",
  "token_type": "Bearer"
}
```

#### レスポンスパラメータ

| パラメータ   | 説明                             |
| ------------ | -------------------------------- |
| access_token | API リクエストに使用するトークン |
| expires_in   | トークンの有効期限（秒）         |
| scope        | 付与された権限の範囲             |
| token_type   | トークンの種類（通常は"Bearer"） |

## セキュリティ上の重要なポイント

### 1. クライアントシークレットの保護

- 絶対に公開しない
- サーバーサイドでのみ使用
- 環境変数などで安全に管理

### 2. state パラメータの使用

- CSRF 攻撃を防ぐため必ず使用
- 十分にランダムな値を使用
- 検証を必ず行う

### 3. 認可コードの取り扱い

- 一回限りの使用
- 速やかにアクセストークンと交換
- 有効期限は短い（通常数分）

### 4. アクセストークンの管理

- セキュアに保存
- 有効期限の管理
- 必要に応じて更新

## 認可グラントタイプの種類と特徴

### 1. 認可コードグラント

最も一般的で安全な認可フロー。主にサーバーサイドアプリケーション（コンフィデンシャルクライアント）で使用されます。

**特徴**

- アクセストークンとリフレッシュトークンの両方を取得可能
- フロントエンドには認可コードのみを渡し、アクセストークンはサーバーサイドで管理
- 認可コードは一時的なものであり、不正利用された場合のリスクを最小限に抑制

### 2. インプリシットグラント

フロントエンドアプリケーション（パブリッククライアント）向けに設計されたフロー。ただし、セキュリティリスクが高いため現在は非推奨です。

**特徴**

- 認可コードを使用せず、直接アクセストークンを取得
- リダイレクト URL にアクセストークンが露出
- リフレッシュトークンは取得不可

#### インプリシットグラントの実装例

1. **認可リクエスト**

```http
GET https://accounts.google.com/o/oauth2/auth
    ?response_type=token
    &client_id=xxx.apps.googleusercontent.com
    &state=abcd
    &scope=https://www.googleapis.com/auth/calendar.events
    &redirect_uri=http://localhost/callback
```

2. **認可レスポンス**

```http
GET http://localhost/callback
    #access_token=ya29.a0ARW5m75tvsHFVGFS4...
    &token_type=Bearer
    &expires_in=3599
    &scope=https://www.googleapis.com/auth/calendar.events
```

## パブリッククライアントの推奨実装：PKCE

インプリシットグラントに代わる安全な方法として、PKCE を使用した認可コードグラントが推奨されています。

### PKCE の仕組み

PKCE は、認可リクエストとトークン取得リクエストの関連性を証明する仕組みを提供します。

**主要な要素**

1. `code_verifier`：クライアントが生成するランダムな文字列
2. `code_challenge`：code_verifier から生成された検証用の値
3. `code_challenge_method`：変換方法の指定（plain or S256）

### 実装手順

1. **認可リクエスト時**

   - 通常の認可コードグラントのパラメータに加えて
   - `code_challenge`と`code_challenge_method`を追加

2. **トークン取得時**
   - 通常の認可コードグラントのパラメータに加えて
   - `code_verifier`を追加

この方法により、パブリッククライアントでも安全に認可コードグラントを利用できます。

// 認可リクエスト
https://accounts.google.com/o/oauth2/auth?response_type=code&client_id=.apps.googleusercontent.com&state=abcd&scope=https://www.googleapis.com/auth/calendar.events&redirect_uri=http://localhost/callback&access_type=offline

// 認可レスポンス
http://localhost/callback?state=abcd&code=4%2F0AanRRrtCgvq43QVBSjg1CMxpV7YXJ6P3q6JPHyWj9mfKh9QGifXvkvIiEINWP7gwqQujCA&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcalendar.events

// アクセストークン取得リクエスト
// リフレッシュトークンが欲しい
https://oauth2.googleapis.com/token?code=4%2F0AanRRrtCgvq43QVBSjg1CMxpV7YXJ6P3q6JPHyWj9mfKh9QGifXvkvIiEINWP7gwqQujCA&client_id=.apps.googleusercontent.com&client_secret=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&grant_type=authorization_code&redirect_uri=http://localhost/callback

// アクセストークン取得レスポンス
{
"access_token": "ya29.a0ARW5m75dv1bVwOPb5CZJ9cc4_GSPY3STbN4vdJqdq1Iy68Y9yxlrJtvDQgYj0JyudgVd3CjGglApex5-7Fw16oJZR0goNQ_08IP8Pbmo0n9yd88jCSSU0wsm6BHk6CVjQvyaOAz1iS5ZOBR7VLf16LZ4-18kMLSBbL4aCgYKAWwSARMSFQHGX2Mi9SbMX_JjL2cKfbF6Pi7rAA0170",
"expires_in": 3560,
"scope": "https://www.googleapis.com/auth/calendar.events",
"token_type": "Bearer"
}

OpenID Connect とは、OAuth 2.0 の拡張使用であり、認証を目的としたプロトコルである。

OpenID Connect の各種フローについて学習していましたが、
その中で紹介された ハイブリットフローの存在意義がよく分からず 🤔

そもそも、認可コードフローは、フロントエンドに id_token を露出して漏洩するリスクを避けるためのものなのに、ハイブリットフローとして認可コードフローと併せてわざわざフロントエンドに id_token を露出するインプリシットフローを行う必要があるのか？というところ。

現場でハイブリットフローを使用する例はあるのかな？
あるとしたら、どんな利点があるのかな？
