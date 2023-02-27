---
author: "Futo Horio"
title: "Azure Static Web App + Hugo で Web サイトを公開する"
date: "2023-02-26"
Description: ""
hideSummary: true
ShowWordCount: false
tags: ["Azure"]
ShowToc: true
ShowBreadCrumbs: true
cover:
  image: /images/2023-02-26-cover.png
draft: false
---

Azure Static Web App + Hugo を使って、  
静的 Web サイトの作成と公開を行う方法を紹介します。

独自ドメイン (``futo-horio.dev``) は「お名前.com」で新規取得して、  
Azure DNS にネームサーバーを変更した後、  
Azure Static Web App の **[設定] > [カスタムドメイン]** から設定を追加しました。


# 事前準備

- [Hugo](https://github.com/gohugoio/hugo/releases/tag/v0.110.0) をインストール & PATH 設定を行う

# Hugo アプリを新規作成する

Hugo CLI を使って Hugo アプリを新規作成する

```cli
hugo new site docs
cd docs
```

git リポジトリの初期セットアップ

```cli
git init
git branch -M main
```

git サブモジュールとしてテーマ (``PaperMod``) をインストールする

```cli
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
echo theme = 'PaperMod' >> config.toml
```

変更内容をコミットする
```cli
git add -A
git commit -m "Initial commit"
```

ローカル環境で Hugo アプリを起動する

```cli
hugo server
```

# GitHub に Push する

Web ブラウザで [GitHub ページ](https://github.com/new) を開き、 Public リポジトリを新規作成する。 

![](/images/2023-02-27-github.png)

※ 「Add a README file」 のチェックは外しておく。

GitHub リポジトリをリモートとしてローカルリポジトリに追加する

```cli
git remote add origin https://github.com/<アカウント名>/<リポジトリ名>.git
```

ローカルリポジトリを リモート に Push する

```cli
git push --set-upstream origin main
```

# Azure Static Web App にデプロイする

- Azure Static Web App リソースを作成する

![](/images/2023-02-26-azurestaticwebapp-create.png)

| プロパティ | 設定値 |
| --- | --- |
| **デプロイの詳細** | |
| デプロイの種類 | GitHub |
| GitHub アカウント | 自分のアカウント |
| 組織 | 自分 |
| リポジトリ | 先ほど作成したリポジトリ |
| ブランチ | main |
| **ビルドの詳細** | |
| ビルドのプリセット | Hugo |
| アプリの場所 | / (デフォルト) |
| API の場所 | 指定なし |
| 出力先 | public |


リソースが作成されると、指定したブランチ (```main```) に  
GitHub Actions (.github/workflows/xxxxx.yml) が追加され、  
Azure Static Web App への自動デプロイが開始されます。



# GitHub Actions 実行結果
GitHub Actions の実行結果は GitHub ページの  
Actions タブから確認することができます。

![](/images/2023-02-26-github-actions.png)

GitHub Actions のワークフロー定義ファイル (```.yml```)

```githubActions.yml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_BLACK_FIELD_0119EFB00 }}
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for Github integrations (i.e. PR comments)
          action: "upload"
          ###### Repository/Build Configurations - These values can be configured to match your app requirements. ######
          # For more information regarding Static Web App workflow configurations, please visit: https://aka.ms/swaworkflowconfig
          app_location: "/" # App source code path
          api_location: "" # Api source code path - optional
          output_location: "public" # Built app content directory - optional
          ###### End of Repository/Build Configurations ######

  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_BLACK_FIELD_0119EFB00 }}
          action: "close"
```

GitHub Actions でビルドデプロイが成功すると、
Azure Static Web App で作成した静的 Web サイトを確認することができます。

![Azure Static Web App](/images/2023-02-26-cover.png)

# カスタムドメインの設定
この手順はカスタムドメインを設定するための手順です。

1. 独自ドメインを取得する
2. Azure DNS ゾーンを作成する ( Azure Portal )
3. ネームサーバーを Azure DNS に変更する
4. Azure Static Web App の **[設定] > [カスタムドメイン]** から  
  **[Azure DNS のカスタムドメイン]** を選択する
5. **ドメイン名** と **DNSゾーン** を入力する
6. カスタムドメインが追加されるまで待つ (10分程度)

# Refs

- [Hugo サイトを Azure Static Web Apps に発行する](https://learn.microsoft.com/ja-jp/azure/static-web-apps/publish-hugo)