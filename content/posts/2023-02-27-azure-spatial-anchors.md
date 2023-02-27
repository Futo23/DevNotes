---
author: "Futo Horio"
title: "Azure Spatial Anchors 公式チュートリアル検証 & 解説"
date: "2023-02-27"
Description: ""
hideSummary: true
ShowWordCount: false
tags: ["Azure"]
ShowToc: true
ShowBreadCrumbs: true
cover:
    image: /images/2023-02-27-azure-spatial-anchors.png
draft: false
---
![](images/2023-02-27-22-01-20.png)

基本的な手順は、以下公式ドキュメントを参照。

- [Azure Spatial Anchors を使用して新しい HoloLens Unity アプリを作成する詳細な手順](https://learn.microsoft.com/ja-jp/azure/spatial-anchors/tutorials/tutorial-new-unity-hololens-app?tabs=azure-portal)

# 検証環境
- Windows PC ( Windows 11 Pro / OSビルド : ``22621.1265`` )
- Windows 10 SDK ( ``10.18362.0`` )
- Unity ``2020.3.13f1``
- Mixed Reality Feature Tools ( ``v1.0.2203.0 Preview`` )
  - Azure Spatial Anchors SDK Core ``2.13.3``
  - Azure Spatial Anchors SDK for Windows ``2.13.3`` 
  - Mixed Reality OpenXR Plugin ``1.7.0``
- Visual Studio 2022 ( ``Version 17.1.3`` )
- Microsoft HoloLens 2
- Azure サブスクリプション

# 成果物
HoloLens 2 で Azure Spatial Anchors のクラウドアンカーを作成して、アプリケーション (内部ストレージ) に保存した ``AnchorId`` を元にクラウドアンカーを検索する簡単なアプリケーションです。

| 操作 | 処理内容 |
| --- | --- |
| 任意の場所をタップ ( short tap )  | ASA セッションを開始、手の位置でクラウドアンカーを作成 |
| アンカーをタップ ( short tap ) | ASA クラウドアンカーを削除 |
| 任意の場所をタップ ( long tap ) | ASA セッション停止 & クラウドアンカーの可視化に用いた GameObject を削除 |
| 任意の場所をタップ ( long tap ) | ローカル保存された ``Anchor Id`` を元にクラウドアンカーを検索、<br/>GameObject としてクラウドアンカーを可視化 |

※ Azure Spatial Anchors では GPS や Wi-Fi、BLE ビーコンの情報を元に、クラウドアンカーを検出する Coarse relocalization という機能もありますが、このチュートリアルでは Azure Spatial Anchors のアンカーIDを元にクラウドアンカーを検出します。

- [粗い再局在化](https://learn.microsoft.com/ja-jp/azure/spatial-anchors/concepts/coarse-reloc)

# 作業手順

チュートリアル内の作業手順はざっくり以下の通りです。


1. Unity プロジェクトの新規作成
2. ビルドプラットフォームの変更
3. Azure Spatial Anchors と OpenXR のインポート
4. Unity プロジェクトのセットアップ
5. Azure Spatial Anchors アカウントの作成
6. アプリケーションの作成

# 解説
チュートリアル内のソースコードについて解説していきたいと思います。  
コードの可読性は下がりますが、ソースコードに解説コメントを入れました。

```csharp
using System;
using System.Linq;
using System.Collections;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.XR;
using Microsoft.Azure.SpatialAnchors;
using Microsoft.Azure.SpatialAnchors.Unity;

public class AzureSpatialAnchorsScript : MonoBehaviour
{
    /// <summary>
    /// ショートタップとロングタップを区別するための配列
    /// _tappingTimer[0] : 右手のタイマー
    /// _tappingTimer[1] : 左手のタイマー
    /// </summary>
    private float[] _tapTimer = { 0, 0 };

    /// <summary>
    /// Azure Spatial Anchors インターフェース (SpatialAnchorManager) を格納するメンバ変数
    /// </summary>
    private SpatialAnchorManager _spatialAnchorManager = null;


    /// <summary>
    /// 新規作成、もしくは検索したアンカーを可視化する GameObject を格納するメンバ変数
    /// </summary>
    private List<GameObject> _foundOrCreatedAnchorGameObjects = new List<GameObject>();

    /// <summary>
    /// 新規作成したアンカーの ID を格納するメンバ変数
    /// </summary>
    private List<String> _createdAnchorIDs = new List<String>();

    /// <summary>
    /// GameObject 開始時に実行される処理
    /// </summary>
    void Start()
    {
        // 同一ゲームオブジェクトにアタッチされている SpatialAnchorManager コンポーネントを取得
        _spatialAnchorManager = GetComponent<SpatialAnchorManager>();
        // デバッグログとエラーログをサブスクライブ
        _spatialAnchorManager.LogDebug += (sender, args) => Debug.Log($"ASA - Debug: {args.Message}");
        _spatialAnchorManager.Error += (sender, args) => Debug.LogError($"ASA - Error: {args.ErrorMessage}");
        // SpatialAnchorManger の AnchorLocated コールバックをサブスクライブする
        _spatialAnchorManager.AnchorLocated += SpatialAnchorManager_AnchorLocated;
    }

    /// <summary>
    /// 毎フレーム呼び出される処理
    /// </summary>
    void Update()
    {
        for (int i = 0; i < 2; i++)
        {
            // 入力デバイス (右手 or 左手) を取得
            // InputDevices : XR 入力サブシステムのデバイスへアクセスするためのインターフェース
            // InputDevices.GetDeviceAtXRNode : 指定された XR ノードで入力デバイスを取得 (i == 0 なら右手、そうでなければ左手を取得)
            // [refs] https://docs.unity3d.com/ja/2021.1/ScriptReference/XR.InputDevice.html
            InputDevice device = InputDevices.GetDeviceAtXRNode((i == 0) ? XRNode.RightHand : XRNode.LeftHand);
            // tryGetFeatureValue(CommonUsages.primaryButton) : タップ状態を取得して、isTapping に格納
            // [refs] https://docs.unity3d.com/ja/2021.1/ScriptReference/XR.InputDevice.TryGetFeatureValue.html
            if (device.TryGetFeatureValue(CommonUsages.primaryButton, out bool isTapping))
            {
                if (!isTapping)
                {
                    // タップタイマーが 0f より大きく、1.0f 未満の場合はショートタップと判定
                    if (0f < _tapTimer[i] && _tapTimer[i] < 1.0f)
                    {
                        // タップされた位置を取得して、handPosition に格納
                        if (device.TryGetFeatureValue(CommonUsages.devicePosition, out Vector3 handPosition))
                        {
                            // ショートタップ処理関数を実行
                            ShortTap(handPosition);
                        }
                    }
                    _tapTimer[i] = 0;
                }
                else
                {
                    // タイマーに経過時間を加算
                    _tapTimer[i] += Time.deltaTime;
                    // タイマーが 2.0f 以上の場合はロングタップと判定
                    if (_tapTimer[i] >= 2.0f)
                    {
                        if (device.TryGetFeatureValue(CommonUsages.devicePosition, out Vector3 handPosition))
                        {
                            // ロングタップ処理関数を実行
                            LongTap();
                        }
                        // タイマーをリセット
                        _tapTimer[i] = -float.MaxValue;
                    }
                }
            }
        }
    }

    /// <summary>
    /// ショートタップされた際の処理
    /// </summary>
    /// <param name="handPosition">タップされた位置</param>
    private async void ShortTap(Vector3 handPosition)
    {
        // Azure Spatial Anchors セッション開始
        await _spatialAnchorManager.StartSessionAsync();

        if (!IsAnchorNearBy(handPosition, out GameObject anchorGameObject))
        {
            // Azure Spatial Anchors クラウドアンカーの作成
            await CreateAnchor(handPosition);
        }
        else
        {
            // 近くにあるアンカーオブジェクトを削除
            DeleteAnchor(anchorGameObject);
        }
    }

    /// <summary>
    /// ロングタップされた際の処理
    /// </summary>
    private async void LongTap()
    {
        // Azure Spatial Anchors セッションが開始しているかどうかを確
        if (_spatialAnchorManager.IsSessionStarted)
        {
            // Azure Spatial Anchors セッション停止
            _spatialAnchorManager.DestroySession();
            // アンカーオブジェクトの全削除
            RemoveAllAnchorGameObject();
            Debug.Log("ASA - Stopped Session and remved all anchor Objects");
        }
        else
        {
            // アプリケーション実行中に作成した Azure Spatial Anchors アンカーを検索する
            await _spatialAnchorManager.StartSessionAsync();
            LocatedAnchor();
        }
    }

    /// <summary>
    /// Azure Spatial Anchors アンカーを作成する関数
    /// </summary>
    /// <param name="position">Azure Spatial Anchors が作成される位置</param>
    /// <returns>Async Task</returns>
    private async Task CreateAnchor(Vector3 position)
    {
        // アンカーオブジェクトの作成
        // デバイスの位置を取得 (もし取得できない場合は Vector3.zero を headPosition に格納)
        if (!InputDevices.GetDeviceAtXRNode(XRNode.Head).TryGetFeatureValue(CommonUsages.devicePosition, out Vector3 headPosition))
        {
            headPosition = Vector3.zero;
        }

        // 頭部の位置に合わせてアンカーオブジェクトを回転させる
        Quaternion orientationTowardsHead = Quaternion.LookRotation(position - headPosition, Vector3.up);

        // Cube をアンカーオブジェクトして作成
        GameObject anchorGameObject = GameObject.CreatePrimitive(PrimitiveType.Cube);
        anchorGameObject.GetComponent<MeshRenderer>().material.shader = Shader.Find("Legacy Shaders/Diffuse");
        anchorGameObject.transform.position = position;
        anchorGameObject.transform.rotation = orientationTowardsHead;
        anchorGameObject.transform.localScale = Vector3.one * 0.1f;

        // アンカーオブジェクトに CloudNativeAnchor コンポーネントをアタッチ
        CloudNativeAnchor cloudNativeAnchor = anchorGameObject.AddComponent<CloudNativeAnchor>();
        // cloudNativeAnchor コンポーネントの NativeToCloud() メソッドを実行
        await cloudNativeAnchor.NativeToCloud();
        CloudSpatialAnchor cloudSpatialAnchor = cloudNativeAnchor.CloudAnchor;
        // クラウドアンカーの有効期限を設定
        cloudSpatialAnchor.Expiration = DateTimeOffset.Now.AddDays(0.5);

        // アンカーを保存するために環境データを収集する
        // ※ HoloLens を利用する場合、既にキャプチャされた環境データがある場合は初回呼び出し時でも true を返す
        while (!_spatialAnchorManager.IsReadyForCreate)
        {
            // 環境データの収集の進捗情報
            float createProgress = _spatialAnchorManager.SessionStatus.RecommendedForCreateProgress;
            Debug.Log($"ASA - Move your devices to capture more environment data: {createProgress:0%}");
        }

        Debug.Log($"ASA - Saving cloud anchor...");

        try
        {
            // Azure Spatila Anchors クラウドアンカーを作成する
            await _spatialAnchorManager.CreateAnchorAsync(cloudSpatialAnchor);
            bool saveSucceeded = cloudSpatialAnchor != null;
            if (!saveSucceeded)
            {
                Debug.Log($"ASA - Failed to save, but no exception was thrown.");
                return;
            }

            Debug.Log($"ASA - Save cloud anchor with ID: {cloudSpatialAnchor.Identifier}");
            // アンカーオブジェクトをメンバ変数に格納
            _foundOrCreatedAnchorGameObjects.Add(anchorGameObject);
            // クラウドアンカーID をメンバ変数に格納
            _createdAnchorIDs.Add(cloudSpatialAnchor.Identifier);
            // ゲームオブジェクトカラーを緑に変更
            anchorGameObject.GetComponent<MeshRenderer>().material.color = Color.green;
        }
        catch (Exception exception)
        {
            Debug.Log("ASA - Failed to save anchor: " + exception.ToString());
            Debug.LogException(exception);
        }
    }

    /// <summary>
    /// アンカーオブジェクトの削除
    /// </summary>
    private void RemoveAllAnchorGameObject()
    {
        // ゲームオブジェクトを削除
        foreach (GameObject anchorGameObject in _foundOrCreatedAnchorGameObjects)
        {
            Destroy(anchorGameObject);
        }
        // アンカー情報を可視化するためのゲームオブジェクト情報を削除
        _foundOrCreatedAnchorGameObjects.Clear();
    }

    /// <summary>
    /// _createdAnchorIDs に格納されているクラウドアンカーIDを元に、クラウドアンカーを検索する
    /// </summary>
    private void LocatedAnchor()
    {
        if (_createdAnchorIDs.Count > 0)
        {
                // ストアされた全てのアンカーIDを検出する Watcher を作成
                // Watcher が開始すると、指定された条件に適合するアンカーが見つかった際にコールバックが発生します。
                Debug.Log($"ASA - Creating watcher to look for {_createdAnchorIDs.Count} spatial anchors.");
                AnchorLocateCriteria anchorLocateCriteria = new AnchorLocateCriteria();
                anchorLocateCriteria.Identifiers = _createdAnchorIDs.ToArray();
                _spatialAnchorManager.Session.CreateWatcher(anchorLocateCriteria);
                Debug.Log($"ASA - Watcher created!");
        }
    }

    /// <summary>
    /// アンカーが見つかった際に呼び出される関数
    /// </summary>
    /// <param name="sender">Callback sender</param>
    /// <param name="args">Callback AnchorLocatedEventArgs</param>
    private void SpatialAnchorManager_AnchorLocated(object sender, AnchorLocatedEventArgs args)
    {
        Debug.Log($"ASA - Anchor recognized as a possible anchor {args.Identifier} {args.Status}");

        // コールバックのイベントステータスが Located の場合、アンカーが見つかったことを意味します。
        if (args.Status == LocateAnchorStatus.Located)
        {
            // ゲームオブジェクトの作成と調整はメインスレッドで実行する必要があります。 UnityDispatcher を使用してメインスレッドで実行します。
            UnityDispatcher.InvokeOnAppThread(() =>
            {
                // クラウドアンカー情報を取得
                CloudSpatialAnchor cloudSpatialAnchor = args.Anchor;

                // アンカーゲームオブジェクトの作成
                GameObject anchorGameObject = GameObject.CreatePrimitive(PrimitiveType.Cube);
                anchorGameObject.transform.localScale = Vector3.one * 0.1f;
                anchorGameObject.GetComponent<MeshRenderer>().material.shader = Shader.Find("Legacy Shaders/Diffuse");
                anchorGameObject.GetComponent<MeshRenderer>().material.color = Color.blue;

                // アンカーゲームオブジェクトを Azure Spatial Anchor にリンクさせる
                anchorGameObject.AddComponent<CloudNativeAnchor>().CloudToNative(cloudSpatialAnchor);
                _foundOrCreatedAnchorGameObjects.Add(anchorGameObject);
            });
        }
    }

    /// <summary>
    /// クラウドアンカーの削除
    /// </summary>
    private async void DeleteAnchor(GameObject anchorGameObject)
    {
        // アンカーオブジェクトにアタッチされている CloudNativeAnchor コンポーネントを取得
        // CloudNativeAnchor で管理している CloudSpatialAnchor を取得
        CloudNativeAnchor cloudNativeAnchor = anchorGameObject.GetComponent<CloudNativeAnchor>();
        CloudSpatialAnchor cloudSpatialAnchor = cloudNativeAnchor.CloudAnchor;

        Debug.Log($"ASA - Deleting cloud anchor: {cloudSpatialAnchor.Identifier}");

        // AnchorManager の DeleteAnchorAsync メソッドを使って、クラウドアンカーを削除
        await _spatialAnchorManager.DeleteAnchorAsync(cloudSpatialAnchor);

        // メンバー変数から アンカーID、アンカーゲームオブジェクトを削除
        _createdAnchorIDs.Remove(cloudSpatialAnchor.Identifier);
        _foundOrCreatedAnchorGameObjects.Remove(anchorGameObject);
        // アンカーゲームオブジェクトを廃棄
        Destroy(anchorGameObject);
        
        Debug.Log($"ASA - Cloud anchor deleted!");
    }

    /// <summary>
    /// アンカーが近くにあるかどうかを判定
    /// </summary>
    private bool IsAnchorNearBy(Vector3 position, out GameObject anchorGameObject)
    {
        anchorGameObject = null;

        // 新規作成もしくは、検出したアンカーオブジェクトがない場合は処理終了
        if (_foundOrCreatedAnchorGameObjects.Count <= 0)
        {
            return false;
        }

        // _foundOrCreateAnchorGameObjects から 距離 と 近くのオブジェクト(15cm以内) を取得 
        var (distance, closestObject) = _foundOrCreatedAnchorGameObjects.Aggregate(
            new Tuple<float, GameObject>(Mathf.Infinity, null),
            (minPair, gameobject) =>
            {
                Vector3 gameObjectPosition = gameobject.transform.position;
                float distance = (position - gameObjectPosition).magnitude;
                return distance < minPair.Item1 ? new Tuple<float, GameObject>(distance, gameobject) : minPair;
            });
        
        // 15cm 以内にアンカーオブジェクトが存在するかどうか
        // 存在する場合: true / anchorGameObject に closestObject を代入
        // 存在しない場合: false
        if (distance <= 0.15f)
        {
            anchorGameObject = closestObject;
            return true;
        }
        else
        {
            return false;
        }
    }

}
```

# 注意点
```AzureSpatialAnchorsScript``` をアタッチする GameObject が他の GameObject の子オブジェクトの場合、親オブジェクトの Transform によってアンカーの挙動が変わってしまうので注意が必要。具体的には、親オブジェクトの Transform を (0, 0, 1.2) に設定していたため、アンカー作成時には手元に表示されていたアンカーオブジェクトが検索時には、アプリ起動時の正面方向 1.2m 奥に表示されてしまっていた。

