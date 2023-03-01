---
author: "Futo Horio"
title: "Azure Remote Rendering Showcase App で コンパイルエラー: Exception: IL2CPP error for method 'System.Void POpusCodec.OpusDecoderAsync が発生した際の回避策"
date: "2023-03-01"
Description: ""
hideSummary: true
ShowWordCount: false
tags: ["Azure"]
ShowToc: true
ShowBreadCrumbs: true
draft: false
---

Azure Remote Rendering の Showcase App を試そうと、公式 GitHub リポジトリからソースコードを Clone して試していたんですが、Unity Editor でビルドを実行すると、以下のコンパイルエラーが発生しました。

# エラー内容

```
Exception: IL2CPP error for method 'System.Void POpusCodec.OpusDecoderAsync`1::DataCallbackStatic(System.IntPtr,System.IntPtr,System.Int32,System.Boolean)' in C:\Users\horio\src\tutorials\azure-remote-rendering\Unity\Showcase\App\Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusDecoder.cs:154
System.ArgumentOutOfRangeException: Specified argument was out of the range of valid values.
```

コンパイルエラーの発生箇所。

``Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusDecoder.cs:154``

Photon Voice SDK 内の OpusDecoder.cs 内の 154 行目

![](/images/2023-02-28-photon-voice-decoder.png)

Photon Voice 2 の公式ページの Known Issues を見ても、  
エラー内容は載ってないようでした。

そこで Azure Remote Rendering の公式リポジトリの Issue を確認すると、
19時間前に Close したばかりの Issue を発見しました。

- [Failing to compile Showcase app #93](https://github.com/Azure/azure-remote-rendering/issues/93)

# 解決策
解決策は、エラーの該当メソッドをコメントアウトすることです。  
今回のコンパイルエラーの原因は Azure Remote Rendering SDK 側ではなく、  
Photon Voice SDK 内のソースコードが原因なので、回避策としてはこれくらいしかないとのこと。

以下、2つの Photon Voice スクリプトのファイルを修正します。

Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusDecoder.cs

```Showcase\App\Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusDecoder.cs
//[AOT.MonoPInvokeCallbackAttribute(typeof(Action<IntPtr, IntPtr, int, bool>))]
static public void DataCallbackStatic(IntPtr handle, IntPtr p, int count, bool endOfStream)
{
	//if (handles.TryGetValue(handle, out var obj))
	//{
	//    obj.dataCallback(p, count, endOfStream);
	//}
}
```

Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusEncoder.cs

```Showcase\App\Assets\Photon\PhotonVoice\PhotonVoiceApi\Core\POpusCodec\OpusEncoder.cs
//[AOT.MonoPInvokeCallbackAttribute(typeof(Action<IntPtr, IntPtr, int>))]
static public void DataCallbackStatic(IntPtr handle, IntPtr p, int count)
{
	//if (handles.TryGetValue(handle, out var obj))
	//{
	//    obj.dataCallback(p, count);
	//}
}
```

修正後、Unity Editor でビルドを実行すると、  
エラーは発生せずにビルドが成功しました👏
