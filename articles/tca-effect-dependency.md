---
title: "TCA の Effect と Dependency で、保存処理を Reducer から切り離してテストする"
emoji: "💾"
type: "tech"
topics: ["swift", "swiftui", "tca", "ios"]
published: false
---

## はじめに

TCA で保存処理を Effect で実行し、その中身を Dependency で差し替えられるようにすると、UserDefaults に書かずに保存の成功と失敗をテストできます。この記事では学習内容を保存する画面を例に、その手順をまとめます。

:::message
TCA 1.26.2 で書いています。
:::

## Effect：保存処理は Reducer の外で動かす

Reducer は Action を受け取って State を変え、最後に Effect を返します。保存のように Reducer の外で動かしたい処理は Effect の中に書き、結果は Action として Reducer に戻します。

Reducer が返す Effect には、次の2つを使いました。

- `return .none`：ほかに何もしない
- `return .run { send in ... }`：非同期の処理を動かし、終わったら `send` で次の Action を送る

保存の状態は State の `saveStatus` に持っています。

| 値 | 状態 |
|----|------|
| `.idle` | 初期値 |
| `.saving` | 保存中 |
| `.saved` | 保存できた |
| `.failed` | 保存に失敗した |

最初は保存したことにして1秒待つだけの Effect を書きました。

```swift:LearningFeature.swift
case .saveButtonTapped:
    state.saveStatus = .saving
    return .run { send in
        try? await Task.sleep(for: .seconds(1))
        await send(.saveCompleted)
    }

case .saveCompleted:
    state.saveStatus = .saved
    return .none
```

保存ボタンを押すと、まず `saveStatus` が `.saving` になります。Effect が1秒待ってから `.saveCompleted` を送り、それを受け取った Reducer が `saveStatus` を `.saved` にします。ただし、このコードは実際には何も保存しません。動きを確かめるための仮の実装です。

## Dependency：保存の中身を Reducer から切り離す

次に、仮の実装を本物の保存処理に置き換えます。保存先には UserDefaults を使います。保存する学習内容は State の `learningContent` に入っています。

Effect の中で `UserDefaults.standard.set` を直接呼ぶと、次のようになります。

```swift
case .saveButtonTapped:
    state.saveStatus = .saving
    return .run { [content = state.learningContent] send in
        do {
            try validateContent(content)  // 空ならエラーを投げる
            UserDefaults.standard.set(content, forKey: "learningContent")
            await send(.saveCompleted)
        } catch {
            await send(.saveFailed)
        }
    }
```

アプリはこれでも動きます。ただ、保存処理が Reducer に直接書かれているので、テストで Reducer を動かすと本物の UserDefaults に書き込まれます。そこで、保存と読み込みを `LearningClient` にまとめ、テストのときだけ中身を差し替えられるようにしました。

```swift:LearningFeature.swift
struct LearningClient {
    var save: @Sendable (String) async throws -> Void
    var load: @Sendable () async -> String?
}
```

`save` と `load` は保存と読み込みの処理を表すクロージャです。`@Sendable` は Effect のように別のタスクから呼んでも安全なクロージャであることを示す印です。ここでは、保存先が UserDefaults なのか、別の仕組みなのかを決めていません。

アプリを動かすときに使う中身（`liveValue`）では、UserDefaults に保存しています。

```swift:LearningFeature.swift
// DependencyKey に準拠すると、Dependency として登録できる
extension LearningClient: DependencyKey {
    // アプリで使う中身。型は LearningClient 自身
    static let liveValue = LearningClient(
        save: { content in
            try validateContent(content)
            UserDefaults.standard.set(content, forKey: "learningContent")
        },
        load: {
            return UserDefaults.standard.string(forKey: "learningContent")
        }
    )
}

// learningClient という名前で登録する
extension DependencyValues {
    var learningClient: LearningClient {
        get { self[LearningClient.self] }
        set { self[LearningClient.self] = newValue }
    }
}
```

`validateContent` は、学習内容が空なら `SaveError.emptyContent` を投げる関数です。

```swift:LearningFeature.swift
enum SaveError: Error {
    case emptyContent
}

func validateContent(_ content: String) throws {
    if content.isEmpty {
        throw SaveError.emptyContent
    }
}
```

Reducer では `LearningFeature` のプロパティとして `client` を用意し、Effect の中で呼びます。

```swift:LearningFeature.swift
@Reducer
struct LearningFeature {
    @Dependency(\.learningClient) var client
    // ...
}
```

`client` には、アプリでは `liveValue` が、テストでは `withDependencies` で差し替えた中身が入ります。

最初はなぜ `DependencyValues` に名前を足すのかわかりませんでした。足しておくと、Reducer では `@Dependency(\.learningClient)`、テストでは `$0.learningClient.save` と、同じ名前で書けます。名前を足さずに、`@Dependency(LearningClient.self)` と型を直接渡す書き方もあります。

保存処理は次のようになりました。

```swift:LearningFeature.swift
case .saveButtonTapped:
    state.saveStatus = .saving
    return .run { [content = state.learningContent] send in
        do {
            try await client.save(content)
            await send(.saveCompleted)
        } catch {
            await send(.saveFailed)
        }
    }
```

直接呼ぶ版と比べると、`validateContent` と `UserDefaults.standard.set` の2行は `liveValue` に移り、Reducer には `client.save(content)` の1行だけが残りました。Reducer は、保存先が UserDefaults であることを知りません。読み込みも同じように、画面が出たときの `.onAppear` で `client.load()` を呼んでいます。

## テストで保存の中身を差し替える

テストでは `withDependencies` で保存の中身を差し替えられます。そのため、実際に UserDefaults に書かずに、保存ボタンを押したら `.saveCompleted` が届くことを確かめられます。`$0.learningClient` の `learningClient` も `DependencyValues` に足した名前です。

`store.send` では、ボタンを押した直後の State の変化を確かめます。`store.receive` では、Effect から届いた Action と、その Action による State の変化を確かめます。

### 保存に成功する場合

`save` をエラーを投げずに終わる中身に差し替え、`.saving` から `.saved` になることを確かめます。

```swift:tcaTests.swift
let store = TestStore(
    initialState: LearningFeature.State(learningContent: "TCA勉強")
) {
    LearningFeature()
} withDependencies: {
    $0.learningClient.save = { content in
        #expect(content == "TCA勉強")
    }
}

await store.send(.saveButtonTapped) {
    $0.saveStatus = .saving
}
await store.receive(.saveCompleted) {
    $0.saveStatus = .saved
}
```

テスト用の `save` では `#expect` で、渡された学習内容が `"TCA勉強"` かを確かめています。

### 保存に失敗する場合

`save` をエラーを投げる中身に差し替え、`.saving` から `.failed` になることを確かめます。

```swift:tcaTests.swift
let store = TestStore(
    initialState: LearningFeature.State(learningContent: "")
) {
    LearningFeature()
} withDependencies: {
    $0.learningClient.save = { content in
        throw SaveError.emptyContent
    }
}

await store.send(.saveButtonTapped) {
    $0.saveStatus = .saving
}
await store.receive(.saveFailed) {
    $0.saveStatus = .failed
}
```

`withDependencies` の中で、`save` を `throw SaveError.emptyContent` だけの中身に差し替えています。この `save` は渡された内容に関係なく必ずエラーを投げます。

すると、Reducer の保存処理は次のように進みます（`.saveButtonTapped` の Effect から、必要なところだけ抜き出しています）。

```swift:LearningFeature.swift
do {
    try await client.save(content)  // ここでエラーが出る
    await send(.saveCompleted)
} catch {
    await send(.saveFailed)         // catch に入り、.saveFailed を送る
}

// ...

case .saveFailed:
    state.saveStatus = .failed              // .saveFailed を受け取り、.failed にする
    return .none
```

このテストで確かめているのは、保存の中身がエラーを投げたときの Reducer の動きです。`validateContent` が空の文字列でエラーを投げることや、UserDefaults への書き込みが成功することは確かめていません。

## おわりに

保存処理を `LearningClient` に切り出したことで、Reducer は保存先を知らずに済み、テストでは保存の中身を成功する版と失敗する版に差し替えて確かめられるようになりました。
