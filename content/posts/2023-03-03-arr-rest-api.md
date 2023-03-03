---
author: "Futo Horio"
title: "Azure Remote Rendering セッション経過時間を取得する"
date: "2023-03-03"
Description: ""
hideSummary: true
ShowWordCount: false
tags: ["MixedReality"]
ShowToc: true
ShowBreadCrumbs: true
draft: false
---

Azure Remote Rendering の REST API を使って、特定のセッションのセッションステータスを取得する方法を調査してみました。

# 前提条件
- Azure サブスクリプションを持っていること
- Azure Remote Rendering リソースを作成済みであること

# Remote Rendering - REST API
Azure Remote Rendering では、以下8種類の API が提供されています。

| 項目 | 概要 |
| --- | --- |
| API Version | 2021-01-01 |
| API Docs | https://docs.microsoft.com/en-us/rest/api/mixedreality/2021-01-01/remote-rendering |
| **API 一覧** | 
| Create Conversion | Azure Blob Storage アカウントに保存されたアセットを ArrAsset に変換する API |
| Create Session | 新しいレンダリングセッションを作成可能な API |
| Get Conversion | 特定のコンバージョン (モデル変換) スタータスを取得可能な API |
| Get Session | 特定のセッションのプロパティを取得する API |
| List Conversions | コンバージョン ( モデル変換 ) 一覧を取得する API |
| List Sessions | リモートレンダリングセッション一覧を取得する API |
| Stop Session | 特定のリモートレンダリングセッションを停止する API |
| Update Session | 特定のリモートレンダリングセッションの接続時間 (最大値) を更新する API  |

# 実現したいこと
過去に作成したセッションの 経過時間 と VM サイズ を取得したい。  

※ List Sessions API で取得できるかと思いきや、"開始中" または "準備完了" 状態のセッション一覧でした。

# 実現方法
Azure Remote Rendering セッション作成時に生成される UUID を使って、  
Get Session API を呼び出すことで、終了済みのセッションの経過時間 や VM サイズを取得することができます。

# 試すこと

1. テスト用のセッションを作成する
2. セッションID (UUID) を元に Get Session API を呼び出す
3. セッションを停止する
4. セッションID (UUID) を元に Get Session API を呼び出す

**1. テスト用のセッションを作成する**

``https://remoterendering.japaneast.mixedreality.azure.com/accounts/{accountId}/sessions/{sessionId}?api-version=2021-01-01-preview`` (PUT)

**リクエストヘッダー**

| キー | 値 |
| --- | --- |
| Authorozation | Bearer {accessToken} |
| Content-Type | application/json |

**リクエストボディ**
```
{
    maxLeaseTimeMinutes: 5,
    size: "standard"
}
```

**レスポンス**

```
{
    "id":"20230303-sample-session",
    "creationTime":"2023-03-03T13:35:18.697407+00:00",
    "elapsedTimeMinutes":0,
    "maxLeaseTimeMinutes":5,
    "size":"Standard",
    "status":"Starting"
}
```

**2. セッションID (UUID) を元に Get Session API を呼び出す**

``https://remoterendering.japaneast.mixedreality.azure.com/accounts/{accountId}/sessions/{sessionId}?api-version=2021-01-01-preview`` (GET)

**リクエストヘッダー**

| キー | 値 |
| --- | --- |
| Authorozation | Bearer {accessToken} |

**レスポンス**

```
{
    "id": "20230303-sample-session",
    "creationTime": "2023-03-03T13:35:18.697407+00:00",
    "arrInspectorPort": 54686,
    "handshakePort": 54856,
    "elapsedTimeMinutes": 1,
    "hostname": "xxxxxxxxxxx.remoterendering.vm.japaneast.mixedreality.azure.com",
    "maxLeaseTimeMinutes": 5,
    "size": "Standard",
    "status": "Ready",
    "teraflops": 8.1
}
```

**3. セッションを停止する**

``https://remoterendering.japaneast.mixedreality.azure.com/accounts/{accountId}/sessions/{sessionId}/:stop?api-version=2021-01-01-preview`` (POST)

**リクエストヘッダー**

| キー | 値 |
| --- | --- |
| Authorozation | Bearer {accessToken} |

**レスポンス**  
204 No Content

**4. セッションID (UUID) を元に Get Session API を呼び出す**

``https://remoterendering.japaneast.mixedreality.azure.com/accounts/{accountId}/sessions/{sessionId}?api-version=2021-01-01-preview`` (GET)

**リクエストヘッダー**

| キー | 値 |
| --- | --- |
| Authorozation | Bearer {accessToken} |

**レスポンス**

```
{
    "id": "20230303-sample-session",
    "creationTime": "2023-03-03T13:35:18.697407+00:00",
    "elapsedTimeMinutes": 3,
    "maxLeaseTimeMinutes": 5,
    "size": "Standard",
    "status": "Stopped",
    "teraflops": 8.1
}
```

セッションの基本情報は、30日間保持されており、セッション終了後も API 経由で取得可能です。※ただし、UUID (セッションID) が不明だと取得できないので、保存する必要があります。

> ![](/images/2023-03-03-arr-session-info.png)
> [Remote Rendering のセッション](https://learn.microsoft.com/ja-jp/azure/remote-rendering/concepts/sessions) より抜粋。


# Refs
- [Remote Rendering | REST API](https://learn.microsoft.com/ja-jp/rest/api/mixedreality/2021-01-01preview/remote-rendering/get-session?tabs=HTTP)
- [Azure Remote Rendering REST API を使って Azure Blob Storage の3Dモデルを変換する方法](https://qiita.com/Futo_Horio/items/8714547ea7346f548cea)