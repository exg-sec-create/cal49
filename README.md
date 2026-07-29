# 第49期 年間休日カレンダー

## ページ構成

- `index.html`: Google ログイン後に休日・有給・行事を編集する管理者（編集者）ページです。
- `staff.html`: 管理者が保存したカレンダーを読み込む閲覧専用ページです。編集機能はありません。

管理者ページの「統計」から「部署別カレンダーを印刷・PDF保存」を選ぶと、各カレンダーを1ページずつ配布用に出力できます。
両ページのヘッダーから相互に移動できます。管理者ページの「設定」では、管理画面へのログインを許可するGoogleアカウントのメールアドレスを追加・削除できます。

main

## Google ログインの設定

管理画面 (`index.html`) は Google Identity Services を使用しています。ログイン画面で
`Error 400: origin_mismatch` が表示される場合、Firebase や Supabase の問題ではなく、
Google Cloud の OAuth クライアントに現在のサイトの生成元が登録されていません。

1. [Google Cloud Console](https://console.cloud.google.com/apis/credentials) を開く。
2. `index.html` の `GOOGLE_CLIENT_ID` と同じ「ウェブ アプリケーション」OAuth 2.0
   クライアントを開く。
3. **承認済みの JavaScript 生成元**に、ログイン画面下部に表示される生成元を追加する。
   例: `https://example.github.io`（パス、末尾の `/` は含めない）。
4. 保存し、設定反映後にブラウザを再読み込みする。

生成元はスキーム・ホスト・ポートが完全一致する必要があります。本番 URL とローカル開発
URL (`http://localhost:PORT`) はそれぞれ個別に登録してください。「承認済みのリダイレクト
URI」ではなく「承認済みの JavaScript 生成元」への登録が必要です。

### 切り分け

- Google の画面に遷移する前に `origin_mismatch` になる: 上記 OAuth 生成元設定。
- Google ボタンが出ない: `accounts.google.com/gsi/client` のブロック、通信、ブラウザ設定。
- 認証後に「アクセス権限がありません」になる: `index.html` の `ALLOWED_EMAILS`。
- 認証後に保存できない: Supabase の URL、キー、RLS ポリシーを確認する。

> OAuth クライアント ID は公開される識別子であり HTML 内に置けますが、クライアント
> シークレットは絶対にコミットしないでください。
