---
author: "Futo Horio"
title: "Azure Static Web App + Hugo で Web サイトを公開する"
date: "2023-02-26"
Description: ""
hideSummary: false
ShowWordCount: false
tags: ["Azure", "Hugo", "Static Web App"]
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


# 手順

**[事前準備]** Hugo CLI をインストール

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

# GitHub に Push する

Web ブラウザで GitHub を開き、 Public リポジトリを新規作成しておく。  
※ README.md は作成しないこと。

GitHub リポジトリをリモートとしてローカルリポジトリに追加する

```cli
git remote add origin https://github.com/<アカウント名>/<リポジトリ名>.git
```

ローカルリポジトリを リモート に Push する

```cli
git push --set-upstream origin main
```

| オプション | 説明 |
| --- | --- |
| --set-upstream | リモートリポジトリを追跡するように設定する |

# Azure Static Web Apps にデプロイする

- Azure Static Web App リソースを作成する

![](/images/2023-02-26-azurestaticwebapp-create.png)

リソースが作成されると、指定したブランチ (```main```) に  
GitHub Actions (.github/workflows/xxxxx.yml) が追加され、  
Azure Static Web App への自動デプロイが開始されます。

# GitHub Actions 実行結果
GitHub Actions の実行結果は GitHub ページの  
Actions タブから確認することができます。

![](/images/2023-02-26-github-actions.png)

# カスタムドメインの設定

1. 独自ドメインを取得する
2. Azure DNS ゾーンを作成する ( Azure Portal )
3. ネームサーバーを Azure DNS に変更する
4. Azure Static Web App の **[設定] > [カスタムドメイン]** から  
  **[Azure DNS のカスタムドメイン]** を選択する
5. **ドメイン名** と **DNSゾーン** を入力する
6. カスタムドメインが追加されるまで待つ (10分程度)