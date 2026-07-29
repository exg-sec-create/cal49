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

codex
以下の順番で設定してください。

#### 1. FirebaseプロジェクトとWebアプリを用意する

1. [Firebase Console](https://console.firebase.google.com/) でプロジェクトを作成または選択する。
2. 「プロジェクトの設定」→「全般」→「マイアプリ」でWebアプリ（`</>`）を追加する。
3. 表示されたFirebase SDK設定の `apiKey` を控える。APIキーはWebアプリに含める識別情報であり、
   アクセス制御は後述のAuthenticationとSecurity Rulesで行う。

#### 2. Realtime Databaseを作成する

1. 「ビルド」→「Realtime Database」→「データベースを作成」を選ぶ。
2. 利用者に近いロケーションを選ぶ。
3. 最初は **ロックモード** を選ぶ（テストモードの全公開ルールをそのまま運用しない）。
4. 作成後、データ画面上部のURLを控える。URLはプロジェクトやロケーションにより
   `https://PROJECT_ID-default-rtdb.firebaseio.com` または
   `https://PROJECT_ID.REGION.firebasedatabase.app` の形式になる。

#### 3. GoogleログインをFirebase Authenticationで有効にする

管理画面のGoogleログインと、Realtime Databaseへの書き込み許可は別の設定です。

1. 「ビルド」→「Authentication」→「始める」を選ぶ。
2. 「Sign-in method」から **Google** を有効にし、プロジェクトのサポートメールを設定する。
3. 「Authentication」→「Settings」→「Authorized domains」に公開先のホスト名を追加する。
   例: `example.github.io`。ここにはスキームやパスを含めない。
4. Google Cloud Console側のOAuthクライアントにも、前述の「承認済みのJavaScript生成元」を設定する。

アプリはGoogle Identity ServicesのIDトークンをFirebase Authenticationのトークンに交換し、
管理画面からの保存時だけ認証トークンをRealtime Database REST APIへ送信します。閲覧者ページは
ログインなしで読み込むため、Rulesでは `cal49/shared` の読み取りだけを公開します。

#### 4. Firebase設定ファイルを配置する（Firebase Hosting以外）

1. `firebase-config.example.json` を `firebase-config.json` という名前でコピーする。
2. `apiKey` に手順1のWeb APIキー、`databaseURL` に手順2のURLを設定する。
3. `firebase-config.json` を `index.html`、`staff.html` と同じ場所へ公開する。

```json
{
  "apiKey": "FirebaseのWeb APIキー",
  "databaseURL": "https://PROJECT_ID-default-rtdb.firebaseio.com"
}
```

Firebase Hostingでは `/__/firebase/init.json` がこれらの値を自動提供するため、このファイルは
不要です。サービスアカウント秘密鍵やOAuthクライアントシークレットは絶対に配置しないでください。

#### 5. Realtime Database Security Rulesを設定する

Realtime Databaseの「ルール」タブへ `database.rules.example.json` の内容を貼り付けて
「公開」を押します。サンプルは次の方針です。

- `cal49/shared` の読み取りは閲覧者ページ向けに公開する。
- 書き込みはFirebase Authenticationで認証済みかつ、列挙した管理者メールだけに許可する。
- 保存データに必須項目がない書き込みを拒否する。
- `cal49/shared` 以外への読み書きを拒否する。

管理者を変更する場合、管理画面内の管理者一覧に加えて、Rules内のメールアドレスも追加・削除して
再公開してください。管理画面の一覧だけを変更してもSecurity Rulesの書き込み権限は変わりません。
これは、ブラウザから管理者自身がRulesを書き換えられないようにするためです。

> **重要:** `.write: true` は設定しないでください。HTMLでログイン画面を表示していても、第三者は
> ブラウザを経由せずDatabase URLへ直接リクエストできます。CORS、公開元ドメイン、Google OAuthの
> JavaScript生成元設定もRealtime Databaseの書き込み認可の代わりにはなりません。

#### 6. 動作確認とエラーの見方

1. 閲覧者ページを開き、「最終更新」または「データなし」と表示されることを確認する。
2. 許可されたGoogleアカウントで管理画面へログインし、カレンダーを変更する。
3. 約1秒後に「保存済み」と表示されることを確認する。
4. 閲覧者ページの「更新」を押し、変更が反映されることを確認する。

- `Firebase未設定`: `firebase-config.json` の配置場所、JSON形式、`databaseURL` を確認する。
- `Firebase Authentication用の apiKey が未設定`: `apiKey` を追加する。
- `OPERATION_NOT_ALLOWED`: Firebase AuthenticationでGoogleプロバイダを有効にする。
- `HTTP 401`: Firebase Authentication、APIキー、ログインセッションを確認する。
- `HTTP 403` または `PERMISSION_DENIED`: Security Rulesと管理者メールを確認する。
- ブラウザConsoleにCORSや通信エラー: Database URL、ネットワーク、ブラウザ拡張機能を確認する。

設定後に画面の「同期」または「更新」を押してください。ブラウザの開発者ツールのConsoleにも
接続失敗の理由が出力されます。

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
main
