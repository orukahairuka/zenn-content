---
title: "SwiftUI で RealityView のあとに ARView を出すと真っ黒になることがある"
emoji: "🕶️"
type: "tech"
topics: ["swift", "swiftui", "arkit", "realitykit", "ios"]
published: true
---

## はじめに

ハッカソンで作った AR アプリで、AR の画面がときどき真っ黒になりました。最小構成のアプリで確かめると、`RealityView` を外したあとに作った `ARView` で起きていました。確かめた条件と、2つの回避策を紹介します。

:::message
検証環境：Xcode 27.0 / iPhone 13（iOS 27.2）
:::

## 症状

3D モデルを表示する画面から AR の画面に移ると、ときどき AR の画面が真っ黒になりました。アプリは落ちず、エラーも出ません。直前の画面では `RealityView`（`content.camera = .virtual`）で 3D を表示していました。

## カメラは動いていた

`ARSessionDelegate` でログを出すと、真っ黒の間もカメラの映像は届いていました。

```swift
// frameCount は var frameCount = 0 で宣言しておく
func session(_ session: ARSession, didUpdate frame: ARFrame) {
    frameCount += 1
    if frameCount % 300 == 0 { print("フレーム \(frameCount)") }
}
```

カメラから映像を受け取るところまでは動いているので、`ARView` が画面に表示する側の問題だと考えました。

## 最小構成で検証する

検証用のアプリで条件を変えて試しました。`arView.snapshot` で撮った画面を 16×16 に縮めて輝度を平均し（0〜1）、0.03 未満のときを「真っ黒」としています。カメラ映像（`ARFrame.capturedImage`）の明るさも測り、カメラの前が暗いだけではないことを確かめています。

### RealityView のあとに ARView を作る

前の画面を出して外したあと、`ARView` を作りました。

| 前の画面 | 結果 |
|---|---|
| 2D の画面だけ | ならない |
| `RealityView`（`.virtual`） | **ときどき真っ黒** |
| `SCNView` | ならない |

真っ黒になったのは `RealityView` のあとだけです。外してから 0.5秒待って作っても、`arView.environment.background = .cameraFeed()` を明示しても起きました。真っ黒の間もカメラのフレームは届き、RealityKit のシーンの更新も続いていました。

### 動いている ARView は真っ黒にならない

`ARView` を動かしたまま、ほかの画面を出して戻りました。

| 操作 | 結果 |
|---|---|
| `ARView` の上に `RealityView` を重ねて外す | ならない |
| `fullScreenCover` で別の画面を出して閉じる | ならない |
| `TabView` の別のタブで `RealityView` を出して戻る | ならない |
| `TabView` で、タブを離れるときに `ARView` を外し、戻ったら作り直す | **真っ黒になる** |

タブを離れるときに `ARView` を外す作りでは、戻ったときに `RealityView` のあとで `ARView` を作り直すことになり、真っ黒になりました。

### 一度なると、作り直しても戻らない

真っ黒になったあと、`RealityView` を挟まずに `ARView` を作り直すと、真っ黒のままでした。アプリを起動し直すと戻りました。

## 回避策

内部の原因は分かっていないので、どちらも暫定の回避策です。

### ARView を作り直さない

`ARView` を `RealityView` より先に作り、外さずに残しておくと、真っ黒になりませんでした。`TabView` なら、タブを離れても `ARView` を外さない作りにします。この方法なら `RealityView` をそのまま使えます。

### 3D 表示に SceneKit を使う

カメラ映像のいらない 3D 表示を `RealityView` から SceneKit の `SCNView` に替えると、真っ黒にならなくなりました。

```swift
struct ModelSceneView: UIViewRepresentable {
    func makeUIView(context: Context) -> SCNView {
        let view = SCNView()
        view.backgroundColor = .clear
        view.autoenablesDefaultLighting = true

        let scene = SCNScene()
        // scene.rootNode にモデルとカメラを足す
        view.scene = scene
        return view
    }

    func updateUIView(_ uiView: SCNView, context: Context) {}
}
```

ただし、SceneKit は WWDC25 で非推奨になっています。

## 似た報告

- [RealityView camera feed not shown](https://developer.apple.com/forums/thread/786543)：`.virtual` の `RealityView` のあとに出した AR の画面が真っ黒
- [Showing 3D/AR content on multiple pages in iOS with RealityView](https://developer.apple.com/forums/thread/814818)：先に `RealityView` を使うと、あとの AR の画面でカメラ映像が出ない（`ARView` でも同じ）
- [AR draws the model over a black background](https://github.com/sceneview/sceneview/issues/3912)：最初の画面に `.virtual` の `RealityView` があるアプリで、AR の背景が真っ黒
