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
&client_secret=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
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

# 用語集

## スコープ

クライアントが認可サーバーに要求するアクセストークンの権限範囲を通知するもの。

## 認可コード
