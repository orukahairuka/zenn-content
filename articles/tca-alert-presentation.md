---
title: "TCA でアラートを出して、押されたボタンを Reducer で受け取る"
emoji: "🚨"
type: "tech"
topics: ["swift", "swiftui", "tca", "ios"]
published: true
---

## はじめに

[前の記事](https://zenn.dev/sakurainohirune/articles/tca-basics-in-my-words)では TCA の State・Action・Reducer・Store を整理しました。今回は同じ学習記録の練習アプリに、学習時間をリセットする前の確認アラートを付けました。

TCA ではアラートも State と Action で書きますが、`Action` の中にさらに `Alert` という enum を書く理由が分かりませんでした。この記事ではアラートを出して「リセット」を受け取るまでの書き方と、その理由をまとめます。TCA を触り始めたばかりの人向けです。

:::message
TCA 1.26.2 で書いています。
:::

## アラートを State に置く

TCA ではアラートの中身も State に置きます。`@Presents` を付けた `AlertState` が `nil` でなければアラートが出ます。

```swift:LearningFeature.swift
@ObservableState
struct State: Equatable {
    // ...
    @Presents var alert: AlertState<Action.Alert>? = nil
}

enum Action: Equatable {
    // ...
    enum Alert: Equatable {
        case confirmReset
    }
    case alert(PresentationAction<Alert>)
}
```

画面の「リセット」ボタンが押されたら、Reducer で `AlertState` を作ります。`ButtonState` がアラートのボタンで、`action: .send(.confirmReset)` は「このボタンが押されたら `.confirmReset` を送る」という指定です。

```swift:LearningFeature.swift
case .resetButtonTapped:
    state.alert = AlertState {
        TextState("学習時間をリセットしますか？")
    } actions: {
        ButtonState(role: .cancel) {
            TextState("キャンセル")
        }
        ButtonState(role: .destructive, action: .send(.confirmReset)) {
            TextState("リセット")
        }
    }
    return .none
```

## 「リセット」を押すと何が届くか

アラートの「リセット」を押すと、Reducer には次の値が届きます。

```swift
.alert(.presented(.confirmReset))
```

この値は、enum の中に enum が入った形です。外側から読むと次のとおりです。

| 部分 | どの enum の case か | 意味 |
|---|---|---|
| `.alert(...)` | 自分で書いた `Action` | アラートに関係する Action |
| `.presented(...)` | TCA の `PresentationAction` | アラートのボタンが押された |
| `.confirmReset` | 自分で書いた `Alert` | 押されたのは「リセット」 |

`PresentationAction` は、アラートで Reducer が知りたい2種類のことをまとめた enum です。

- `.presented(Alert)`：Action を設定したボタンが押された。中身はアプリごとに違うので、自分で `Alert` に書く
- `.dismiss`：Action を設定していないボタン（今回は「キャンセル」）などで、アラートが閉じられた

「リセット」を押したときに届くのは `.alert(.presented(.confirmReset))` だけで、`.dismiss` は届きません。アラートが閉じるたびに `.dismiss` が届くわけではありません。

Reducer ではこの形のまま受け取ります。

```swift:LearningFeature.swift
case .alert(.presented(.confirmReset)):
    state.learningTime = 0
    return .none

case .alert(.dismiss):
    return .none
```

`body` の最後には `.ifLet` を付けます。`\.$alert` で State のどこにアラートがあるか、`\.alert` で Action のどの case がアラートのものかを指定します。

```swift:LearningFeature.swift
var body: some ReducerOf<Self> {
    Reduce { state, action in
        // ...
    }
    .ifLet(\.$alert, action: \.alert)
}
```

アラートのボタンの Action や `.dismiss` が届いたとき、`state.alert` を `nil` に戻すのは `.ifLet` です。上の Reducer で `state.alert = nil` と書いていないのは、このためです。

## アラートの Action を Alert に分ける理由

アラート用の Action を、画面の Action とは別の `Alert` に分けて書く理由は2つあります。アラートのボタンに入れる Action を間違えたときにビルドで止められることと、Reducer で画面のボタンの Action と混ざらないことです。

`AlertState` は、TCA が使っている swift-navigation で次のように定義されています。

```swift:AlertState.swift（swift-navigation）
public struct AlertState<Action>: Identifiable {
  public var buttons: [ButtonState<Action>]
  // ...
}
```

`AlertState` の `<>` に書いた型が、そのままボタンの型にも使われます。State に `AlertState<Action.Alert>` と書いたので、ボタンは `ButtonState<Action.Alert>` です。

そのため、ボタンの `.send(...)` に入れられるのは `Action.Alert` の case（`.confirmReset`）だけです。画面の Action（たとえば保存ボタンの `.saveButtonTapped`）を入れると、ビルドできません。

```swift
ButtonState(role: .destructive, action: .send(.saveButtonTapped))
// error: type 'LearningFeature.Action.Alert' has no member 'send'
```

エラー文は `send` がないと言っていますが、原因は `Action.Alert` に `.saveButtonTapped` がないことです。`.confirmReset` に戻すとビルドできます。

2つ目の理由は、Reducer に届く値の形です。画面の「リセット」ボタンで届くのは `.resetButtonTapped`、アラートの「リセット」で届くのは `.alert(...)` で始まる値なので、同じ「リセット」でも Reducer で混ざりません。

なお、`Alert` を `Action` の中に書く必要はありません。`Action` の外に `enum ResetAlertAction { case confirmReset }` と書き、`AlertState<ResetAlertAction>` と `PresentationAction<ResetAlertAction>` にしてもビルドできます。

## View にアラートを出す

アラートのタイトルとボタンは、Reducer で `AlertState` に書いています。そのため View には TCA が用意している `.alert` を付ける1行だけを書きます。`$store` を使うので、View の `store` には `@Bindable` を付けます。

```swift:LearningView.swift
struct LearningView: View {
    @Bindable var store: StoreOf<LearningFeature>

    var body: some View {
        NavigationStack(/* ... */) {
            // ...
        }
        .alert($store.scope(state: \.alert, action: \.alert))
    }
}
```

TCA のソースを読むと、この `.alert` は次のように動きます。

| 起きること | `.alert` がすること |
|---|---|
| `state.alert` が `nil` でなくなる | アラートを出す。タイトルとボタンは `AlertState` から作る |
| 「リセット」が押される | Store に `.alert(.presented(.confirmReset))` を送る |
| 「キャンセル」などで閉じる | `state.alert` がまだ残っていれば、Store に `.alert(.dismiss)` を送る |

「リセット」を押したときは、先に `.ifLet` が `state.alert` を `nil` にしているので、閉じても `.dismiss` は送られません。Reducer に `case .alert(.dismiss)` を書いているのは、「キャンセル」で閉じたときにこの Action が届くからです。

## テストで確かめる

アラートが State にあるので、`TestStore` で次の流れを確かめられます。`TestStore` では画面のボタンを実際に押すのではなく、ボタンを押したときに届く Action を `send` で直接送り、Reducer の動きを確かめます。

1. 学習時間が100分の状態で、画面の「リセット」ボタンを押すと、`state.alert` に `AlertState` が入る
2. アラートの「リセット」を選ぶと、`state.alert` が `nil` に戻り、学習時間（`learningTime`）が0になる

```swift:tcaTests.swift
let store = TestStore(
    initialState: LearningFeature.State(learningTime: 100)
) {
    LearningFeature()
}

await store.send(.resetButtonTapped) {
    $0.alert = AlertState {
        TextState("学習時間をリセットしますか？")
    } actions: {
        ButtonState(role: .cancel) {
            TextState("キャンセル")
        }
        ButtonState(role: .destructive, action: .send(.confirmReset)) {
            TextState("リセット")
        }
    }
}

await store.send(.alert(.presented(.confirmReset))) {
    $0.alert = nil
    $0.learningTime = 0
}
```

2つ目の `send` で `$0.alert = nil` を書いているのは、`.ifLet` がアラートを `nil` に戻すからです。

## まとめ

- アラートの中身は `AlertState` として State に置き、画面の「リセット」ボタンの Action で作る
- アラートの「リセット」は `.alert(.presented(.confirmReset))` として Reducer に届く。「キャンセル」で閉じたときは `.alert(.dismiss)` が届く
- `.ifLet` がアラートを `nil` に戻すので、Reducer で `state.alert = nil` を書かなくてよい
- アラートの Action を `Alert` に分けると、ボタンに入れる Action を間違えたときにビルドで止められ、画面のボタンの Action とも混ざらない
