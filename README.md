# Flutter GitHub Actions Template

Flutter の Android/iOS 開発用の GitHub Actions と関連ファイルのテンプレートです

# 主な機能

- `check.yml`
  - push のタイミングで flutter analyze と test を実行します
  - [Problem Matchers](https://github.com/actions/toolkit/blob/main/docs/problem-matchers.md)、[Danger action](https://github.com/marketplace/actions/danger-action) を使って、analyze が出力する `(info|warning|error)` を GitHub の `File chaged` に表示します
- `bump.yml`
  - GitHub の画面上からアプリのバージョン（`pubspec.yaml` の `version:`）をインクリメント（更新）するワークフローです
  - 更新対象の `major.minor.patch(build number)` を選択することが可能です
  - バージョンの更新と合わせて Tag, Release（releaset note）を作成します
    - [Automatically generated release notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes)
  - 更新内容は自動的に push されます
- `bump-pull-request.yml`
  - アプリバージョンの更新を含む release ブランチと Pull Request を作成します
    - `releases/1.0.0+1` のようなブランチを作成します
  - チーム開発や [Protected branch](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) を使っている環境を想定したワークフローです
- `tagging-when-merged.yml`
  - 上述の release ブランチがマージされたタイミングで Tag を作成します
- `deliver.yml`
  - Tag の push イベントをトリガーに Android と iOS のリリービルドとストアへのアップロードを行います
  - つまり `bump.yml` の実行もしくは `bump-pull-request.yml` で作成したリリースブランチがマージされると deliver.yml が実行されます

その他の機能

- dependabot.yml
  - pubspec.yml に含まれるパッケージと GitHub Actions に含まれる action の更新をチェックして PR を作成するための [dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file) の定義です

# セットアップ手順

## ファイル

- `.github`, `Dangerfile`, `Gemfile` をプロジェクトにコピーします
- `.github/workflows` 内の `bump.yml` と `bump-pull-request.yml` を開いて次の値を変更します

  | キー           | 内容                 |
  | -------------- | -------------------- |
  | GIT_USER_EMAIL | Git で使用する email |

## GitHub Actions へ secret を登録

- 以下の情報をプロジェクトの secret に登録します

### 共通

| キー   | 内容                                                                                                                                                    | 取得方法                                                                                                                                                          |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GH_PAT | Personal Access Token、ワークフローで発生したイベントをトリガーにして別のワークフローを実行するために必要（[詳細](#patpersonal-access-token-について)） | [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) |

### Android 用

| キー                                   | 内容                                                                                          | 取得方法                                                                                |
| -------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| ANDROID_KEY_JKS_BASE64                 | keystore ファイル（Base64）                                                                   | [Build and release an Android app Flutter](https://docs.flutter.dev/deployment/android) |
| ANDROID_STORE_PASSWORD                 | keystore のパスワード                                                                         |                                                                                         |
| ANDROID_KEY_ALIAS                      | key alias                                                                                     |                                                                                         |
| ANDROID_KEY_PASSWORD                   | key password                                                                                  |                                                                                         |
| GOOGLE_SERVICE_ACCOUNT_KEY_JSON_BASE64 | Google(GCP) サービスアカウント、ビルドしたバイナリを Google Play にアップロードするために必要 | 参考 https://zenn.dev/altiveinc/articles/how-to-google-service-account                  |

### iOS 用

| キー                            | 内容                                                                                 | 取得方法                                                                                                                      |
| ------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| APPLE_APP_PASS                  | `altool` を使ってビルドしたバイナリを App Store Connect にアップロードするために必要 | [App 用パスワードを使って Apple ID で App にサインインする - Apple サポート (日本)](https://support.apple.com/ja-jp/HT204397) |
| APPLE_APPLE_ID                  | App 用パスワードの AppleID                                                           |                                                                                                                               |
| IOS_CERTIFICATE_P12_BASE64      | 配布用証明書（Base64）                                                               | 参考 https://zenn.dev/pressedkonbu/articles/254ca2fc3cd1ab                                                                    |
| IOS_CERTIFICATE_P12_PASSWORD    | 証明書のパスワード                                                                   |                                                                                                                               |
| IOS_PROVISIONING_PROFILE_BASE64 | 配布用プロビジョニングプロファイル（Base64）                                         |                                                                                                                               |

### ファイルを base64 に変換して secrets に登録する手順

- [Encrypted secrets - GitHub Docs](https://docs.github.com/ja/actions/security-guides/encrypted-secrets#storing-base64-binary-blobs-as-secrets)

## ExportOptions.plist の作成（iOS）

- iOS アプリの App Store 向けのビルドで `ExportOptions.plist` が必要なので以下の手順で取得する
  - Xcode から Product > Archive > Export を選択
  - 初回は App Store Connect に登録するアプリ名を入力
    - App Store Connect にアプリが登録される
  - Manual managing signing を選択する
    - あらかじめ作成しておいた配布用プロビジョニングプロファイルを選択する
  - Export を実行
    - 出力先のフォルダに入っている `ExportOptions.plist` をプロジェクトの `ios/` に追加する
- 参考
  - https://qiita.com/uhooi/items/a17a5d0e5dd5a76191ac
- 補足
  - Xcode のプロジェクト設定で **Automatic manage signing** をオフ、配布用プロビジョニングプロファイルを選択しておく必要があります

## pubspec.yaml

- `pubspec.yaml`のバージョン番号の更新に [cider](https://pub.dev/packages/cider) を使っているので `dev_dependencies` への追加が必要です

```yaml
dev_dependencies:
  cider: ^0.1.1
```

- analyze の rule は [pedantic_mono](https://pub.dev/packages/pedantic_mono) がおすすめです

# 注意事項

セキュリティ関連での補足です

## GitHub Actions のセキュリティについて

- GitHub Actions のセキュリティガイド
  - [Security guides - GitHub Docs](https://docs.github.com/en/actions/security-guides)
- ワークフローの `permissions` については必要最低限のものを使用するように定義しているつもりですが、過不足があれば教えてもらえると助かります
  - [Workflow syntax for GitHub Actions - GitHub Docs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions)

## Personal Access Token(PAT) について

- ワークフローで `GITHUB_TOKEN` を使って発生したイベントから新しいワークフローは実行されません
  - 例えば push イベントで実行するワークフローがあったとします、そのワークフローの中からレポジトリに commit を push したとします、すると push イベントが発生するのでまたワークフローが実行されます、という無限ループが発生するからです
  - [Triggering a workflow - GitHub Enterprise Cloud Docs](https://docs.github.com/en/enterprise-cloud@latest/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow)
- `deploy.yml` は `bump.yml` や `tagging-when-merged.yml` が push した tag のイベントをトリガーに実行されますが、tag の push に `GITHUB_TOKEN` を使用するとワークフローが実行されないため、これを回避するために `PAT` を使っています
- PAT の使い方を誤ると予期しない動作が起きることがあるので注意が必要です
  - PAT が使えない、もしくは利用したくない場合は `workflow run` を使うように改変することで同等のことは実現が可能かと思います
  - [Github Actions の workflow run について](https://zenn.dev/keitacoins/articles/2a715be45e874f)
- なお PAT を作成する際は最近新しく追加された **fine-grained personal access token** がお勧めです、アクセス対象のレポジトリ、権限を細かく制御できます
  - [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-fine-grained-personal-access-token)
  - ここで紹介しているワークフローで使用する際は Permissions で Contents に write 権限を与えてください
- 従来形式(legacy)のアクセストークンを利用する場合は repository に対する write 権限があれば動作すると思います（動作未確認です）

# その他

- `timeout-minutes:` のタイムアウト時間はプロジェクト合わせて調整してください

# リンク

- [Build and release an Android app | Flutter](https://docs.flutter.dev/deployment/android)
- [Build and release an iOS app | Flutter](https://docs.flutter.dev/deployment/ios)
- [アプリへの署名  |  Android デベロッパー  |  Android Developers](https://developer.android.com/studio/publish/app-signing)
- [Dart analyzer の出力を GitHub のファイル上に表示する](https://itome.team/blog/2022/06/dart-analyzer-problem-matcher/)
- [Danger action](https://github.com/marketplace/actions/danger-action)

## GitHub

- [Automatically generated release notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes)
- [Protected branch](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches)
- [Installing an Apple certificate on macOS runners for Xcode development - GitHub Docs](https://docs.github.com/en/actions/deployment/deploying-xcode-applications/installing-an-apple-certificate-on-macos-runners-for-xcode-development)
- base64 [Encrypted secrets - GitHub Docs](https://docs.github.com/ja/actions/security-guides/encrypted-secrets#storing-base64-binary-blobs-as-secrets)
- persmissions [Workflow syntax for GitHub Actions - GitHub Docs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions)
- [Triggering a workflow - GitHub Enterprise Cloud Docs](https://docs.github.com/en/enterprise-cloud@latest/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow)
- [Security guides - GitHub Docs](https://docs.github.com/en/actions/security-guides)
- access token [Creating a personal access token - GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-fine-grained-personal-access-token)
- [Problem Matchers](https://github.com/actions/toolkit/blob/main/docs/problem-matchers.md)
- [Configuration options for the dependabot.yml file - GitHub Docs](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file)
- [dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file)

# 免責

LICENSE は MIT です、自己責任でご利用ください
