# 第49期 年間休日カレンダー

- `index.html`: Googleログインが必要な編集ページ
- `staff.html`: Firebase上のカレンダーを表示する公開・閲覧専用ページ

## セキュリティ上の前提

Firebase Web APIキー、Firebase `databaseURL`、Google OAuthクライアントIDは、ブラウザへ配信されるため**秘密情報ではありません**。値をHTMLから別ファイルへ移しても、利用者は開発者ツールで確認できます。データへのアクセス制御は、Firebase AuthenticationとRealtime Database Security Rulesで必ず実施してください。

一方、OAuthクライアントシークレット、サービスアカウント秘密鍵、秘密鍵を含むJSON、管理用アクセストークンは秘密情報です。これらはこの静的サイトに配置せず、Gitにもコミットしないでください。

現在の設計では、`cal49/shared` はスタッフページのために公開読み取り、書き込みはFirebase Authenticationで認証された許可メールだけに限定します。カレンダーに個人情報や社外秘情報を保存しないでください。非公開データが必要な場合は、スタッフページにも認証を追加してRulesの `.read` を認証必須に変更してください。

## Firebase設定

### Firebase Hosting

Firebase Hostingの `/__/firebase/init.json` を利用するため、追加ファイルは不要です。

### その他の静的ホスティング／ローカル開発

```sh
cp firebase-config.example.json firebase-config.json
```

`firebase-config.json` の `apiKey` と `databaseURL` を対象環境の値に変更します。このファイルは `.gitignore` の対象です。公開時にはHTMLと同じ階層へデプロイしてください。なお、デプロイされた値そのものは公開情報になります。

Firebase Consoleでは次を設定してください。

1. AuthenticationでGoogleプロバイダを有効化する。
2. AuthenticationのAuthorized domainsへ公開ホストを登録する。
3. Google Cloud ConsoleでOAuthクライアントの「承認済みのJavaScript生成元」を限定する。
4. Google Cloud ConsoleのAPIキー制限で、HTTPリファラーを本番オリジンに限定し、利用APIもFirebaseで必要なAPIだけに限定する。
5. Realtime Databaseへ `database.rules.example.json` を公開する。`.write: true` で運用しない。
6. App Checkも有効化し、不正クライアントからの濫用に対する追加防御とする（Rulesの代替にはなりません）。

管理者を変更するときは、`index.html` の初期許可リストと `database.rules.example.json` の書き込み許可メールを同時に更新し、Rulesを再公開してください。ブラウザ側の許可リストだけでは書き込みを保護できません。

## 漏えい時／今回の移行時に行うこと

過去にコミットされたFirebase Web APIキーはGit履歴に残るため、ファイルを削除するだけでは無効化されません。次を実施してください。

1. Google Cloud Consoleで該当キーの利用状況を確認する。
2. 未制限なら、まず本番のHTTPリファラーと必要APIに制限する。
3. 不審な利用がある、または同じキーをサーバー用途にも流用した場合はキーをローテーションする。
4. OAuthクライアントシークレットやサービスアカウント鍵を入力した可能性があれば、直ちに失効・再発行する（Web APIキーとは扱いが異なります）。
5. Realtime DatabaseのRules、監査ログ、課金アラートを確認する。

必要なら履歴から値を消去できますが、既に取得済みのクローンからは消せないため、履歴書き換えだけでなく失効・ローテーションを優先してください。

## 認証実装について

Google IDトークンとFirebase IDトークンは、ページのメモリ内だけに保持し、`localStorage` / `sessionStorage` には永続化しません。Firebaseへの書き込みではトークンをURLクエリへ入れず、`Authorization: Bearer` ヘッダーで送信します。

以前の共通PINは、ブラウザへ配信され公開データにも保存されるため認証要素になりませんでした。印刷機能は既にGoogleログイン後の画面内にあるため、共通PINを廃止しています。

## 動作確認

ローカルサーバーで配信し、許可アカウントで編集・保存できること、未許可アカウントや未認証RESTリクエストでは書き込みが拒否されること、スタッフページでは読み取りだけできることを確認してください。
