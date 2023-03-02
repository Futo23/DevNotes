---
author: "Futo Horio"
title: "Azure Remote Rendering - 概要"
date: "2023-03-02"
Description: ""
hideSummary: true
ShowWordCount: false
tags: ["Azure"]
ShowToc: true
ShowBreadCrumbs: true
draft: false
---

# 概要
Azure Remote Rendering は、高品質でインタラクティブな 3D コンテンツを Azure 上の ハイエンド GPU マシンでレンダリングして、HoloLens 2 や PC などにリアルタイムでストリーム配信できるサービス。

## サーバーのサイズ

| サーバーサイズ | プリミティブの制限 |
| --- | --- |
| Standard | 2000万 |
| Premium | 制限無し |

※ プリミティブは、単一の三角形 (三角メッシュ) 、単一の点 (点群メッシュ) のこと。  
※ Premium サーバーは制限がありませんが、描画コンテンツがサービスのレンダリング機能を超えた場合、パフォーマンスが低下する可能性があるので注意が必要です。

- [プリミティブの制限](https://learn.microsoft.com/ja-jp/azure/remote-rendering/reference/vm-sizes#primitive-limits)

## サポートされているモデル形式

- 三角形メッシュ
  - FBX
  - GLTF/GLB
- 点群 ( Point Cloud )
  - XYZ
  - PLY
  - E57
  - LAS、LAZ

※ LAS / LAZ フォーマットは、Azure Remote Rendering SDK ``1.1.23`` 以降のバージョンでサポートが開始されました (2022/10/20)

[サポートされているソース形式](https://learn.microsoft.com/ja-jp/azure/remote-rendering/how-tos/conversion/model-conversion#supported-source-formats)

## ネットワーク要件
インターネット接続では、Azure Remote Rendering のシングルユーザーセッションで一貫して少なくとも **40 Mbps 下り / 5 Mbps 上り** の通信速度が必要です。

[ネットワーク接続に関するガイドライン](https://learn.microsoft.com/ja-jp/azure/remote-rendering/reference/network-requirements#guidelines-for-network-connectivity)

# 価格
価格は使用する VM サイズ、リージョンによって異なります。  
(下の表の価格は 2023年3月2日時点のもの)

![](/images/2023-03-02-arr-price.png)

- [Azure Remote Rendering の価格](https://azure.microsoft.com/ja-jp/pricing/details/remote-rendering/)


# Azure CLI
今後更新予定。

# Refs

- [Azure Remote Rendering ドキュメント](https://learn.microsoft.com/ja-jp/azure/remote-rendering/) 
- [GitHub - Azure/azure-remote-rendering](https://github.com/Azure/azure-remote-rendering)
- [Azure Remote Rendering 公式サンプル徹底解説](https://speakerdeck.com/futo23/azure-remote-rendering-gong-shi-sanpuruche-di-jie-shuo)
- [MR Dev Days Japan 前夜祭 - Azure Remote Rendering のご紹介](https://speakerdeck.com/futo23/mr-dev-days-japan-qian-ye-ji-azure-remote-rendering-falsegoshao-jie)
- [Azure Remote Rendering Recap - サービス概要と活用事例](https://speakerdeck.com/futo23/azure-remote-rendering-recap-sabisugai-yao-tohuo-yong-shi-li)
