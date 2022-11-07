[https://github.com/pointfreeco/swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture)

[https://mockapi.io/](https://mockapi.io/)

[https://app.quicktype.io/](https://app.quicktype.io/)

# Counter App

### App

```swift
import SwiftUI
import ComposableArchitecture

// 도메인 + 상태
struct CounterState: Equatable {
    var count = 0
    
}

// 도메인 + 액션
enum CounterAction: Equatable {
    case addCount // 카운트를 더하는 액션
    case subtractCount // 카운트를 빼는 액션
}

struct CounterEnvironment {
    
}

let counterReducer = AnyReducer<CounterState, CounterAction, CounterEnvironment> { state, action, environment in
    switch action {
    case .addCount:
        state.count += 1
        return Effect.none
    case .subtractCount:
        state.count -= 1
        return Effect.none
    }
}

@main
struct TCA_ARCHITECTURE_TUTORIALApp: App {
    
    let counterStore = Store(initialState: CounterState(),
                             reducer: counterReducer,
                             environment: CounterEnvironment())
    
    var body: some Scene {
        WindowGroup {
            CounterView(store: counterStore)
        }
    }
}
```

### CounterView

```swift
import SwiftUI
import ComposableArchitecture

struct CounterView: View {
    
    let store: Store<CounterState, CounterAction>

    var body: some View {
        WithViewStore(self.store) { viewStore in
            VStack {
                Text("\(viewStore.state.count)")
                HStack {
                    Button("Add", action: {
                        viewStore.send(.addCount)
                    })
                    Button("Subtract", action: {
                        viewStore.send(.subtractCount)
                    })
                }
            }
        }
    }
}
```

---

# Memo App

- Async Call

### App

```swift
import SwiftUI
import ComposableArchitecture

@main
struct TCA_ARCHITECTURE_TUTORIALApp: App {
    
    let memoStore = Store(initialState: MemoState(),
                          reducer: memoReducer,
                          environment: MemoEnvrionment(memoClient: MemoClient.live,
                                                       mainQueue: .main))
    
    var body: some Scene {
        WindowGroup {
            MemoView(store: memoStore)
        }
    }
}
```

### MemoClient

```swift
import Foundation
import ComposableArchitecture

//https://636938cb15219b849612d66c.mockapi.io/todos

struct MemoClient {
    // single item
    var fetchMemoItem: (_ id: String) -> Effect<Memo, Failure>
    // all items
    var fetchMemos: () -> Effect<Memos, Failure>
   
    struct Failure: Error, Equatable {}
}

extension MemoClient {
    static let live = Self(
        fetchMemoItem: { id in
            Effect.task {
                let (data, _) = try await URLSession.shared.data(from: URL(string: "https://636938cb15219b849612d66c.mockapi.io/todos/\(id)")!)
                return try JSONDecoder().decode(Memo.self, from: data)
            }
            .mapError { _ in Failure() }
            .eraseToEffect()
        }, fetchMemos: {
            Effect.task {
                let (data, _) = try await URLSession.shared.data(from: URL(string: "https://636938cb15219b849612d66c.mockapi.io/todos/")!)
                return try JSONDecoder().decode(Memos.self, from: data)
            }
            .mapError { _ in Failure() }
            .eraseToEffect()
        }
    )
}
```

### MemoView

```swift
import SwiftUI
import ComposableArchitecture

// 도메인 + 상태
struct MemoState: Equatable {
    var memos: Memos = []
    var selectedMemo: Memo? = nil
    var isLoading: Bool = false
}

// 도메인 + 액션
enum MemoAction: Equatable {
    case fetchItem(_ id: String) // 단일 조회
    case fetchItemResponse(Result<Memo, MemoClient.Failure>) // 단일 조회 응답
    case fetchAll // 모두 조회 액션
    case fetchAllResponse(Result<[Memo], MemoClient.Failure>) // 모두 조회 액션 응답
}

// 환경설정 주입
struct MemoEnvrionment {
    var memoClient: MemoClient
    var mainQueue: AnySchedulerOf<DispatchQueue>
}

// 상태와 액션을 가지고 있는 리듀서
// 액션이 들어왔을 때 상태를 변경하는 부분
let memoReducer = AnyReducer<MemoState, MemoAction, MemoEnvrionment> { state, action, environment in
    switch action {
    case .fetchItem(let memoId):
        enum FetchItemId {}
        state.isLoading = true
        return environment
            .memoClient
            .fetchMemoItem(memoId)
            .debounce(id: FetchItemId.self, for: 0.3, scheduler: environment.mainQueue)
            .catchToEffect(MemoAction.fetchItemResponse)
    case .fetchItemResponse(.success(let memo)):
        state.selectedMemo = memo
        state.isLoading = false
        return Effect.none
    case .fetchItemResponse(.failure):
        state.selectedMemo = nil
        state.isLoading = false
        return Effect.none
    case .fetchAll:
        enum FetchAllId {}
        state.isLoading = true
        return environment
            .memoClient
            .fetchMemos()
            .debounce(id: FetchAllId.self, for: 0.3, scheduler: environment.mainQueue)
            .catchToEffect(MemoAction.fetchAllResponse)
    case .fetchAllResponse(.success(let memos)):
        state.memos = memos
        state.isLoading = false
        return Effect.none
    case .fetchAllResponse(.failure):
        state.isLoading = false
        return Effect.none
    }
}

struct MemoView: View {
    
    let store: Store<MemoState, MemoAction>
    
    var body: some View {
        WithViewStore(self.store) { viewStore in
            List {
                Section(header:
                    VStack(spacing: 8) {
                        Button("Fetch Memo Lists") {
                            viewStore.send(.fetchAll)
                        }
                        Text("Selected Memo info")
                        Text(viewStore.state.selectedMemo?.id ?? "Empty")
                        Text(viewStore.state.selectedMemo?.title ?? "Empty")
                    }
                ) {
                    ForEach(viewStore.state.memos) { memo in
                        Button(memo.title) {
                            viewStore.send(.fetchItem(memo.id))
                        }
                    }
                }
            }
            .listStyle(.plain)
        }
    }
    
}
```
