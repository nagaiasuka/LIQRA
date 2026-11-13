# LIQRA 開発・運用引き継ぎ

LIQRAは写真付きの訪問記録をボードと地図で振り返るアプリです。記録は初期状態で非公開。共通スポットの記録は複数ボードに反映でき、ボードごとに対象を外せます。

## 作業を始める

1. Gitのソース、検証用Supabase、Expo、必要なストア管理画面へのアクセスを運用担当者から受け取る。個人のログインや署名鍵を共有しない。
2. Nodeは `.nvmrc` に従う。iOSのローカルビルドはMacとXcode、AndroidはAndroid StudioとSDKを用意する。
3. ソース一式のあるチェックアウトで以下を実行する。資料だけのチェックアウトでは実行できない。

```sh
nvm use
npm ci
cp .env.example .env
npm run typecheck
npm test
npm run ios
# または npm run android / npm run web
```

`.env` に検証用のSupabase URLとpublishable keyを入れる。設定済みの認証プロバイダーだけを有効にする。Androidの実地図にはアプリID・署名で制限したMapsキーが必要。OAuthのシークレットやDB管理キーはアプリ・Web・Gitに入れない。ネイティブプロジェクトはExpo設定から生成する。設定や依存関係のネイティブ変更後は再ビルドする。

## 構成と現在地

- `app/` はExpo Routerの画面、`src/` は記録・ボード・同期・公開処理、`supabase/migrations/` はDB定義、`tests/` はテスト。認証コールバックは `/auth/callback`。
- Supabase AuthでApple・Google・LINE、DBとStorageで記録・写真を管理。未送信の変更は端末に残し、クラウド保存を確認してから写真の端末コピーを整理する。
- ゲストは「みんな」を閲覧可能。公開には最終確認が必要で、取り下げ後も本人の記録と達成は残る。通報はSupabaseの運用者が確認する。
- iOSシミュレーターで既存記録の保存、別端末相当の復元、公開前確認まで検証済み。実投稿を使う別アカウントのいいね・取り下げの一連操作、Androidの通し確認、ストア提出は未検証。
- 常設のWeb管理画面およびEASのリリース設定は未構築。下記は初回構築手順を含む。

## DBと公式テンプレート

新しい検証DBで、`supabase/migrations/` のSQLをファイル名順に適用する。バックアップ・公開投稿 → クラウド記録 → 公式テンプレート → 通報・ブロックの順。既存環境はDashboardで適用済み定義を照合し、未適用分だけ反映する。過去にSQL Editorから適用した環境では、CLIの履歴と実際の定義が一致するとは限らない。履歴を照合せずに再適用しない。

各プロバイダーのクライアント設定とSupabaseの許可リストを合わせ、モバイルの `liqra://auth/callback` と、使用するWeb環境の `/auth/callback` を登録する。管理者は承認されたLIQRAユーザーのIDを `template_admins` に追加。Webの `/admin/templates` から公式データを初期登録する。以後はテンプレートを公開すればアプリに反映される。既存スポットのIDは変更しない。

本番変更前に検証DBでRLS・保存・旧アプリとの互換性を確認し、復旧点を確保する。DBの変更は先に互換性を維持した状態で適用し、アプリを後で配布する。障害時は配布を止め、互換性のある修正版を出す。DB復元は後から保存された利用者データに影響するため、影響を確認してから判断する。

## Web管理画面のデプロイ

1. 対象環境の公開変数をビルド時に設定し、`npm run export:web` を実行。
2. `dist/` をHTTPS対応の静的ホストへ配信。直接アクセスでも `/admin/templates` と `/auth/callback` がSPAの `index.html` に到達するよう設定する。
3. Supabaseに正確なコールバックURLを追加。未ログイン・一般ユーザーは編集できず、管理者だけ下書き・公開できることを確認。
4. ビルド元のGitコミットとデプロイ先を記録。問題時は前のWeb成果物へ戻す。

## iOS・Androidのストアリリース

1. Expoの所有組織、Apple Developer / App Store Connect、Google Play Consoleの管理権限を用意。初回は `eas build:configure` でEASプロジェクトとビルドプロファイルを設定する。現在のアプリIDは両OSとも `app.liqra.prototype`。初回ストア登録前に正式IDを決定し、変更する場合はOAuth・署名・Maps制限も合わせて更新する。
2. リリース用の公開環境変数をEASへ設定。署名資格情報は管理サービスで扱う。 `app.json` のバージョンと、iOSのビルド番号・AndroidのversionCodeを管理する。同一のビルド番号を再提出しない。
3. 型チェック、テスト、実機確認を行い、公開対象のコミットを特定する。正式なアイコン、ストア画像、説明、プライバシーポリシー、ストアのデータ申告、アカウント削除・投稿管理の対応状況を確認する。
4. `eas build --platform ios --profile production` と `eas build --platform android --profile production` で署名済み成果物を作る。プロファイル作成後のコマンド例。まずTestFlightとPlayの内部テストに配布し、各OSでログイン・写真保存・復元・公開/いいね/取り下げ・通報/ブロックを確認する。
5. 確認したビルドを選んで `eas submit --platform ios` / `eas submit --platform android` でアップロードするか、ストアから手動アップロードする。初回Androidアップロードの条件はPlay ConsoleとExpoの案内を確認。**アップロードと一般公開は別工程**。各ストアで審査を進め、公開日時または段階配信を設定する。
6. Gitコミット、バージョン、ビルド番号、DB適用状況を記録。障害時は段階配信を停止し、修正版を新しい番号で提出する。

参照：[Expo Build](https://docs.expo.dev/build/setup/)・[ストア提出](https://docs.expo.dev/deploy/submit-to-app-stores)・[Supabaseマイグレーション](https://supabase.com/docs/guides/deployment/database-migrations)。
