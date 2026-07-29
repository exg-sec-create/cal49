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
- 認証後に保存できない: Firebase Realtime Database が有効で、`cal49/shared` を管理者ページから読み書きできる Security Rules になっているか確認する。

管理者一覧のクラウド取得に失敗した場合も「確認中」のまま停止せず、5秒後にHTML内の
初期管理者一覧を使ってGoogleログインを開始します。管理画面自体は許可されたGoogle
アカウントで保護されるため共通パスワードは重ねて要求しません。共通PIN `5005` は、
管理画面ログインではなく「部署別カレンダーを印刷・PDF保存」の確認にのみ使用します。

> OAuth クライアント ID は公開される識別子であり HTML 内に置けますが、クライアント
> シークレットは絶対にコミットしないでください。

## Firebase の共有データ

管理者ページと閲覧ページは、Firebase Hosting が提供する `/__/firebase/init.json` から
同じプロジェクトの設定を取得し、Realtime Database の `cal49/shared` にカレンダーを保存・
読み込みします。このため、端末ごとのローカルキャッシュではなく、全員が同じ最新データを
参照できます。ローカルキャッシュは通信できない場合の表示・保存待ち用途にだけ使用します。

### 「クラウド未接続」と表示される場合

Firebase Hosting 以外（GitHub Pages、通常のWebサーバー、ローカルサーバーなど）では
`/__/firebase/init.json` が提供されないため、追加設定が必要です。

1. Firebase Console で **Realtime Database** を作成する。
2. `firebase-config.example.json` を `firebase-config.json` という名前でコピーする。
3. Firebase Console の Realtime Database 画面に表示される URL を `databaseURL` に設定する。
4. `firebase-config.json` を `index.html`、`staff.html` と同じ場所へ公開する。
5. Realtime Database の Security Rules で、利用方法に合った `cal49/shared` の読み書きを許可する。

Firebase Hosting を利用している場合は `firebase-config.json` は不要です。この場合は、Firebase
Console で Realtime Database が作成済みか、プロジェクトに `databaseURL` が設定されているか、
Security Rules によって REST API の読み書きが拒否されていないかを確認してください。

設定後に画面の「同期」または「更新」を押してください。ブラウザの開発者ツールの Console
にも接続失敗の理由が出力されます。なお、Firebase の Web 設定値や `databaseURL` は公開用の
識別情報ですが、サービスアカウント鍵やクライアントシークレットは配置しないでください。
