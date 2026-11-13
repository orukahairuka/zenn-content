---
title: "TCA の画面遷移を StackState で書くまでに詰まったこと"
emoji: "🪜"
type: "tech"
topics: ["swift", "swiftui", "tca", "ios"]
published: true
---

## はじめに

学習時間を記録する練習アプリで、詳細画面への遷移を SwiftUI 標準の `NavigationLink` から TCA の `StackState` に書き換えました。この記事では書き換えた順番と、その途中で分からなかったことをまとめます。

TCA の State・Action・Reducer・Store は[前の記事](https://zenn.dev/sakurainohirune/articles/tca-basics-in-my-words)で整理しています。

:::message
TCA 1.26.2 で書いています。
:::

## SwiftUI 標準の遷移と何が違うのか

最初は SwiftUI 標準の書き方で詳細画面に遷移していました。

```swift:LearningView.swift
NavigationStack {
    NavigationLink("学習詳細を見る") {
        LearningDetailView()
    }
}
```

これでも遷移はできます。ただ、詳細画面が開いているかどうかを TCA の State では管理していません。遷移は SwiftUI が管理していて、Reducer からは見えません。

SwiftUI にも `NavigationStack(path:)` で状態から遷移する書き方はあるので、状態で遷移を管理すること自体は TCA だけの機能ではありません。TCA ではその状態を Feature の State に置きます。そうすると、画面遷移もほかの State と同じように Reducer で変えられます。

## StackState を State に持たせる

TCA では開いている遷移先の画面を `StackState` で持ちます。ルートの学習記録画面は含まず、詳細画面を開くと詳細画面の State がひとつ積まれます。

`StackState` は、配列の `Array<Int>` と同じように、中に入れる型を `<>` で指定します。

```swift
StackState<遷移先の State の型>
```

`LearningFeature.State` に `path` を足します。最初は何も開いていないので、空の `StackState` を入れておきます。

```swift:LearningFeature.swift
var path: StackState<Path.State> = StackState()
```

`Path` は遷移先になりうる画面をまとめる enum で、次の節で作ります。`Path.State` を指定しておくと、`path` には `Path` に書いた画面の State だけを入れられます。

型を右辺に書いて `var path = StackState<Path.State>()` としても同じです。

## 遷移先をまとめる Path を作る

`Path.State` の `Path` は TCA に用意されている型ではなく自分で作る型です。遷移先の画面をまとめる enum で、`@Reducer` を付けます。`@Reducer` は struct だけでなく enum にも付けられます。

まず、`LearningFeature` の中に空の `Path` を書きました。

```swift:LearningFeature.swift
@Reducer
enum Path {

}
```

## 遷移先の LearningDetailFeature を作る

`Path` に登録する前に、遷移先の詳細画面の Feature を別のファイルに作りました。遷移先の画面も自分の State と Action を持ちます。

```swift:LearningDetailFeature.swift
@Reducer
struct LearningDetailFeature {
    @ObservableState
    struct State: Equatable {
        var learningTime: Int = 0
    }

    enum Action {
        case closeButtonTapped
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .closeButtonTapped:
                return .none
            }
        }
    }
}
```

「閉じる」ボタンの処理は後で書くので、いまは何もせず `.none` を返しておきます。

## Reducer の中に Reducer を書いてよいのか

`LearningFeature` の中に `@Reducer` を付けた `Path` を書いたとき、Reducer の中に Reducer を書いてよいのか分かりませんでした。

ここには2つの関係があります。

- `LearningFeature` の中に `Path` を書いているのは、Swift の型を入れ子にしているだけ
- `Path` から `LearningDetailFeature` を使うのが、TCA の Reducer の組み合わせ

`LearningDetailFeature` は別のファイルにあっても問題ありません。それぞれの Reducer は自分の State と Action だけを扱います。

## Path に詳細画面を登録する

`Path` に、遷移先の Feature を case として並べます。

```swift:LearningFeature.swift
@Reducer
enum Path {
    case detail(LearningDetailFeature)
}
```

`Path` の中に State と Action は書いていませんが、`@Reducer` マクロが case をもとに自動で作ります。

## body と View をつなぐ

Action には `path` を足し、`body` の最後に `.forEach` を付けます。

```swift:LearningFeature.swift
@ObservableState
struct State: Equatable {
    // ...
    var path: StackState<Path.State> = StackState()
}

enum Action: Equatable {
    // ...
    case path(StackActionOf<Path>)
}

var body: some ReducerOf<Self> {
    Reduce { state, action in
        switch action {
        // ...
        case .path:
            return .none
        }
    }
    // 1つ目は State の path、2つ目は Action の path
    .forEach(\.path, action: \.path)
}
```

`.forEach` が詳細画面の Action と State を `LearningDetailFeature` の Reducer に渡します。

View では `NavigationLink` に遷移先の View ではなく State を渡します。

```swift:LearningView.swift
// 親の Store から path の部分だけを切り出して渡す
NavigationStack(path: $store.scope(state: \.path, action: \.path)) {
    // 遷移先の State を渡す
    NavigationLink(
        "学習詳細を見る",
        state: LearningFeature.Path.State.detail(
            LearningDetailFeature.State(learningTime: store.learningTime)
        )
    )
    // ...
} destination: { pathStore in
    // pathStore は path に積まれた画面1つ分の Store
    switch pathStore.case {
    case .detail(let detailStore):
        // detailStore は詳細画面の Store
        LearningDetailView(store: detailStore)
    }
}
```

`scope` は Store から一部だけを切り出すメソッドです。`NavigationStack` は切り出した `path` を見て画面を表示し、戻るボタンで戻ったときは `path` から画面を取り除きます。書き換えもするので、`$` を付けて Binding で渡します。

`destination` の `pathStore` はどの画面かがまだ決まっていない Store です。`pathStore.case` で case を調べ、`.detail` なら詳細画面の Store を `detailStore` に取り出して `LearningDetailView` に渡します。

## 詳細画面を閉じる

最初は何もしなかった `closeButtonTapped` では、`@Dependency(\.dismiss)` を使って詳細画面を閉じるようにしました。

```swift:LearningDetailFeature.swift
@Dependency(\.dismiss) var dismiss

case .closeButtonTapped:
    return .run { _ in
        await dismiss()
    }
```

## まとめ

- 開いている画面は State の `path` で持つ
- 遷移先の画面は `Path` にまとめ、`.forEach` で Reducer をつなぐ
- View では `NavigationLink` に State を渡し、`destination` で View を決める
