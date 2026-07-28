---
title: "HTTPの仕組みを基礎から発展まで理解する"
emoji: "🌐"
type: "tech"
topics: ["http", "https", "rest", "graphql", "oauth"]
published: true
---

## はじめに

Webを使っていれば必ず裏側で動いている HTTP の仕組みを、基礎から発展まで体系的にまとめました。「なんとなく知っている」を「ちゃんとわかる」に変えることを目指しています。概念の説明だけでなく、`curl` を使った実際の動作確認も交えて解説します。

---

## 全体像：まずここを掴む

```
あなたのブラウザ  ────リクエスト────▶  サーバー
                  ◀───レスポンス────
```

ブラウザが「このページをください」と**リクエスト**を送り、サーバーが「はい、どうぞ」と**レスポンス**を返す。HTTPはこの1往復を定義したルールです。

---

## ① リクエスト/レスポンスの構造

### リクエストの構造

```
GET /index.html HTTP/1.1       ← メソッド + パス + バージョン
Host: example.com              ← ヘッダー（メタ情報）
Accept: text/html
                               ← 空行（ヘッダーの終わり）
（ボディ：GETの場合は通常なし）
```

### レスポンスの構造

```
HTTP/1.1 200 OK                ← バージョン + ステータスコード
Content-Type: text/html        ← ヘッダー
Content-Length: 1234

<html>...</html>               ← ボディ（実際のデータ）
```

### 重要な特性：ステートレス

HTTPは「記憶しない」プロトコルです。1リクエストごとに接続が独立しており、サーバーは前のリクエストを覚えていません。

たとえば、毎回初めて電話するお店のように、「前回と同じもので」が通じない状態です。だからこそ、「誰が送ったリクエストか」を毎回一緒に伝える仕組みとして **Cookie / Session / JWT** などが生まれました。

---

## ② HTTPメソッド

メソッドは「動詞」、URLは「名詞」として設計します。

```
GET    /users        → ユーザー一覧を取得
POST   /users        → 新しいユーザーを作成
PUT    /users/123    → ID:123のユーザーを全体更新
PATCH  /users/123    → ID:123のユーザーを部分更新
DELETE /users/123    → ID:123のユーザーを削除
```

### 冪等性

| メソッド | 何度実行しても結果が同じ？ |
|---------|--------------------------|
| `GET` | ✅ |
| `PUT` | ✅ |
| `DELETE` | ✅ |
| `POST` | ❌ 叩くたびに新しいデータが増える |
| `PATCH` | ⚠️ 場合による |

ネットワークエラーで再送が必要なとき、冪等なメソッドは安全に再送できます。

### curl での実践

```powershell
# GET
curl.exe -i https://jsonplaceholder.typicode.com/posts/1

# POST
curl.exe -i --request POST https://jsonplaceholder.typicode.com/posts `
  --header "Content-Type: application/json" `
  --data '{"title": "test", "body": "hello", "userId": 1}'

# DELETE
curl.exe -i --request DELETE https://jsonplaceholder.typicode.com/posts/1
```

:::message
PowerShell では `curl` は `Invoke-WebRequest` のエイリアスになっているため、**`curl.exe`** と明示的に書く必要があります。
:::

---

## ③ ステータスコード

番号の「百の位」で意味が決まります。

```
1xx → 処理中
2xx → 成功
3xx → リダイレクト
4xx → クライアントのミス（あなたが悪い）
5xx → サーバーのミス（サーバー側が悪い）
```

### よく使うコード

| コード | 意味 | 使いどころ |
|-------|------|-----------|
| `200 OK` | 成功 | GET・PUTの成功 |
| `201 Created` | 作成成功 | POSTで新規作成 |
| `204 No Content` | 成功・ボディなし | DELETEの成功 |
| `301 Moved Permanently` | 恒久的な移転 | URLが永久に変わった |
| `400 Bad Request` | リクエストがおかしい | パラメータ不足など |
| `401 Unauthorized` | 認証が必要 | ログインしていない |
| `403 Forbidden` | 権限がない | ログイン済みだが権限なし |
| `404 Not Found` | 見つからない | URLが存在しない |
| `500 Internal Server Error` | サーバー内部エラー | バグ・例外発生 |

### 401 と 403 の違い

```
401 → 「誰ですか？」（ログインしていない）
403 → 「あなたには権限がありません」（ログイン済みだが拒否）
```

### エラーを見たときの考え方

```
4xx → 自分のコード（リクエスト）を疑う
5xx → サーバー側を疑う・ログを確認する
```

---

## ④ ヘッダー

ヘッダーは「本文をどう扱うか」の指示書です。

### リクエストヘッダー

```http
Content-Type: application/json       ← ボディの形式を伝える
Authorization: Bearer eyJhbGci...   ← 認証情報
Accept: application/json            ← 受け取りたい形式
Cookie: session_id=abc123           ← セッション情報
```

### レスポンスヘッダー

```http
Content-Type: application/json; charset=utf-8
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
Location: https://example.com/new-page
Cache-Control: max-age=3600
```

### Cookie の重要オプション

| オプション | 意味 |
|-----------|------|
| `HttpOnly` | JavaScriptから読めない（XSS対策） |
| `Secure` | HTTPS通信のときだけ送る |
| `SameSite=Lax` | 別サイトからのリクエストには送らない（CSRF対策） |

### CORS（Cross-Origin Resource Sharing）

ブラウザには「別ドメインへのリクエストをブロックする」セキュリティ機能があります。JavaScriptが別ドメインのデータを読み取る操作を制限するものです。

```
access-control-allow-origin: https://myapp.com  → このドメインを許可
access-control-allow-credentials: true          → 認証情報の送信を許可
access-control-allow-methods: GET, POST         → 許可するメソッド
```

:::message
CORSの設定は**アクセスされる側（APIサーバー）**で行います。`<a>`タグや画像表示はCORSの制限を受けません。JSが裏でデータを読む操作だけが対象です。
:::

---

## ⑤ HTTPS

```
HTTP  → データが「生」のまま流れる（盗み見られる）
HTTPS → データが暗号化されて流れる

HTTPS = HTTP + TLS（暗号化レイヤー）
```

### TLS ハンドシェイクの流れ

```
① ブラウザ → サーバー：「HTTPS通信したい」
② サーバー → ブラウザ：「これが私の証明書です」
③ ブラウザ：証明書を認証局（CA）で検証
④ 共通の鍵を生成して共有
⑤ 以降の通信を暗号化
```

### 証明書とは

「このサーバーは本物です」という第三者機関（CA）のお墨付き。ブラウザにはあらかじめ信頼できるCA一覧が入っており、CAが署名した証明書のみを信頼します。

### 実務で覚えておくこと

```
・本番環境は必ず HTTPS にする
・証明書の無料取得 → Let's Encrypt が定番
・HTTP → HTTPS への強制リダイレクト（301）
・HSTSヘッダーで HTTPS を強制
  Strict-Transport-Security: max-age=31536000
```

---

## ⑥ REST API 設計

RESTはHTTPを正しく・美しく使うための「設計思想」です。

### 基本ルール

**URLは名詞（複数形）、操作はメソッドで表現する**

```
✅ 良い例
GET    /users        → 一覧取得
POST   /users        → 新規作成
GET    /users/123    → 1件取得
PUT    /users/123    → 更新
DELETE /users/123    → 削除

❌ 悪い例
GET /getUsers
POST /createUser
GET /deleteUser?id=123
```

### URL設計のルール

```
・複数形を使う（/users, /products）
・階層で関係を表現する（/users/123/posts）
・小文字・ハイフン区切り（/blog-posts）
・深くしすぎない（3階層が目安）
```

### クエリパラメータの使いどころ

```
GET /products?category=book&sort=price&page=2
```

URLパスは「場所」、クエリは「絞り込み・並び替え・ページネーション」に使います。

### バージョニング

```
/api/v1/users    ← 旧バージョン（動かし続ける）
/api/v2/users    ← 新バージョン（新機能）
```

---

## 発展① GraphQL

### REST との違い

```
REST    → エンドポイントが複数（/users, /posts, ...）
GraphQL → エンドポイントが1つ（POST /graphql）
```

欲しいフィールドをリクエストのボディに書いて指定します。

```graphql
# 名前とメールだけ欲しい
{
  user(id: 1) {
    name
    email
  }
}
```

### REST との比較

```
REST：GET /users/1
→ {"id":1, "name":"田中", "email":"...", "age":30, ...} 全部返る

GraphQL：{ user(id:1) { name } }
→ {"name": "田中"} 欲しいものだけ返る
```

### 使い分け

```
REST      → シンプルなCRUD・一般的なWeb API
GraphQL   → フロントが複数（Web・アプリ等）・取得フィールドを柔軟に変えたい
```

---

## 発展② OAuth 2.0

「Googleでログイン」「GitHubでログイン」の仕組みです。

ここで重要な用語の区別を押さえておきましょう。

```
認証（Authentication） → 「あなたは誰ですか？」を確認すること
認可（Authorization）  → 「あなたに何を許可しますか？」を決めること
```

OAuth はあくまで **認可** の仕組みです。「このアプリにプロフィールを見る権限を与えていいですか？」という許可を管理します。ログインそのもの（認証）は、GitHubやGoogleなど外部サービスが担当します。

### 登場人物

```
リソースオーナー  → あなた（ユーザー）
クライアント      → ログインしたいアプリ（例：Qiita）
認可サーバー      → GitHubの認証システム
リソースサーバー  → GitHubのAPI
```

### Authorization Code フローの流れ

```
① 「GitHubでログイン」をクリック
② Qiita → GitHubの認証画面にリダイレクト
③ GitHubでログイン・権限を許可
④ GitHub → Qiitaに認可コードを渡す
⑤ QiitaサーバーがコードをトークンBと交換
⑥ トークンでGitHub APIを叩いてユーザー情報を取得
```

### スコープ（権限の範囲）

```
scope=read:user    → プロフィールの読み取りだけ
scope=repo         → リポジトリへのアクセス
scope=delete_repo  → リポジトリの削除まで
```

ユーザーが「どの権限を許可するか」を選べるのがOAuthの核心です。

### トークンの種類

```
アクセストークン     → 短命（数時間〜1日）・API呼び出しに使う
リフレッシュトークン → 長命（数週間〜数ヶ月）・トークン再取得に使う
```

---

## 発展③ WebSocket

### HTTP との根本的な違い

```
HTTP
  ブラウザ ──リクエスト──▶ サーバー
  ブラウザ ◀──レスポンス── サーバー（接続終了）

WebSocket
  ブラウザ ──接続──▶ サーバー
  ブラウザ ◀──────────────▶ サーバー（繋ぎっぱなし・双方向）
```

HTTPは「都度電話する」、WebSocketは「電話を繋ぎっぱなし」のイメージです。

### 使いどころ

```
チャット・株価・オンラインゲーム・リアルタイム通知・共同編集
```

### 接続の仕組み

最初だけHTTPでWebSocketへの切り替えを交渉します：

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade

← HTTP/1.1 101 Switching Protocols
```

`101 Switching Protocols` が返ったあとはWebSocketプロトコルで通信します。

### 使い分け

```
更新頻度が低い・取得するだけ → HTTP（REST / GraphQL）
リアルタイム・サーバーから送る → WebSocket
```

---

## curl コマンド早見表

```powershell
# GET（ステータスコードも表示）
curl.exe -i https://example.com/api/users

# POST（JSONを送る）
curl.exe -i --request POST https://example.com/api/users `
  --header "Content-Type: application/json" `
  --data '{"name": "田中", "email": "tanaka@example.com"}'

# PUT（更新）
curl.exe -i --request PUT https://example.com/api/users/1 `
  --header "Content-Type: application/json" `
  --data '{"name": "田中太郎"}'

# DELETE
curl.exe -i --request DELETE https://example.com/api/users/1

# 認証ヘッダー付き
curl.exe -i https://example.com/api/profile `
  --header "Authorization: Bearer YOUR_TOKEN"

# タイムアウト設定
curl.exe -i --max-time 5 https://example.com/api/users
```

:::message
PowerShell では `curl` でなく `curl.exe` を使うこと。シングルクォートでJSONを囲むとバックスラッシュの問題を避けられます。
:::

---

## まとめ

| トピック | 要点 |
|---------|------|
| リクエスト/レスポンス | 1往復が基本単位。ステートレス |
| メソッド | GET取得・POST作成・PUT更新・DELETE削除 |
| ステータスコード | 2xx成功・4xxクライアントエラー・5xxサーバーエラー |
| ヘッダー | Content-Type・Authorization・Cookie・CORS |
| HTTPS | HTTP + TLS。証明書で身元確認・通信を暗号化 |
| REST API | URLは名詞・メソッドで操作・リソース設計 |
| GraphQL | エンドポイント1つ・欲しいフィールドを指定 |
| OAuth 2.0 | パスワードを渡さずにトークンで認可 |
| WebSocket | 双方向リアルタイム通信 |
