# TCA Architecture Reference

The Composable Architecture by Point-Free. TCA 1.x (Swift Concurrency, `@Reducer` macro).

---

## Core Pattern

```
View  ──send(.action)──▶  Reducer  ──returns Effect──▶  async work
  └──reads Store.state        └──mutates State              └──sends back Action
```

- **State**: plain struct, all view-relevant data
- **Action**: enum, every user event and side-effect response
- **Reducer**: pure function `(inout State, Action) → Effect`
- **Store**: runtime that ties it together
- **Effect**: wraps async work, emits Actions back into the system

---

## Basic Feature

```swift
@Reducer
struct ItemListFeature {

    // MARK: - State
    @ObservableState
    struct State: Equatable {
        var items: [Item] = []
        var isLoading = false
        var alert: AlertState<Action.Alert>?
    }

    // MARK: - Action
    enum Action {
        case onAppear
        case itemsLoaded(Result<[Item], Error>)
        case deleteItem(Item)
        case itemDeleted(Result<Void, Error>)
        case alert(PresentationAction<Alert>)

        enum Alert: Equatable {
            case confirmDelete(Item)
        }
    }

    // MARK: - Dependencies
    @Dependency(\.itemService) var itemService

    // MARK: - Reducer Body
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                state.isLoading = true
                return .run { send in
                    await send(.itemsLoaded(Result { try await itemService.fetchItems() }))
                }

            case .itemsLoaded(.success(let items)):
                state.isLoading = false
                state.items = items
                return .none

            case .itemsLoaded(.failure(let error)):
                state.isLoading = false
                state.alert = AlertState { TextState(error.localizedDescription) }
                return .none

            case .deleteItem(let item):
                state.alert = AlertState {
                    TextState("Delete \(item.name)?")
                } actions: {
                    ButtonState(role: .destructive, action: .confirmDelete(item)) {
                        TextState("Delete")
                    }
                    ButtonState(role: .cancel) { TextState("Cancel") }
                }
                return .none

            case .alert(.presented(.confirmDelete(let item))):
                return .run { send in
                    await send(.itemDeleted(Result { try await itemService.delete(item) }))
                }

            case .itemDeleted(.success):
                // Reload or remove locally
                return .send(.onAppear)

            case .itemDeleted(.failure(let error)):
                state.alert = AlertState { TextState(error.localizedDescription) }
                return .none

            case .alert:
                return .none
            }
        }
        .ifLet(\.$alert, action: \.alert)
    }
}
```

---

## View

```swift
struct ItemListView: View {
    @Bindable var store: StoreOf<ItemListFeature>

    var body: some View {
        List(store.items) { item in
            ItemRow(item: item)
                .swipeActions {
                    Button(role: .destructive) {
                        store.send(.deleteItem(item))
                    } label: {
                        Label("Delete", systemImage: "trash")
                    }
                }
        }
        .overlay { if store.isLoading { ProgressView() } }
        .alert($store.scope(state: \.alert, action: \.alert))
        .onAppear { store.send(.onAppear) }
        .navigationTitle("Items")
    }
}

// Preview
#Preview {
    ItemListView(store: Store(initialState: ItemListFeature.State()) {
        ItemListFeature()
    })
}
```

---

## Dependencies

Define dependencies using the `@DependencyClient` macro and register via `DependencyValues`.

```swift
// Dependency client
@DependencyClient
struct ItemServiceClient {
    var fetchItems: @Sendable () async throws -> [Item]
    var delete: @Sendable (Item) async throws -> Void
}

// Register
extension ItemServiceClient: DependencyKey {
    static let liveValue = ItemServiceClient(
        fetchItems: { try await ItemService.shared.fetchItems() },
        delete: { try await ItemService.shared.delete($0) }
    )

    static let testValue = ItemServiceClient(
        fetchItems: { [] },
        delete: { _ in }
    )
}

extension DependencyValues {
    var itemService: ItemServiceClient {
        get { self[ItemServiceClient.self] }
        set { self[ItemServiceClient.self] = newValue }
    }
}

// Use in Reducer
@Dependency(\.itemService) var itemService
```

---

## Composing Features (parent–child)

```swift
@Reducer
struct AppFeature {
    @ObservableState
    struct State {
        var itemList = ItemListFeature.State()
        var path = StackState<Path.State>()
    }

    enum Action {
        case itemList(ItemListFeature.Action)
        case path(StackActionOf<Path>)
    }

    @Reducer
    struct Path {
        enum State { case detail(ItemDetailFeature.State) }
        enum Action { case detail(ItemDetailFeature.Action) }
        var body: some ReducerOf<Self> {
            Scope(state: \.detail, action: \.detail) { ItemDetailFeature() }
        }
    }

    var body: some ReducerOf<Self> {
        Scope(state: \.itemList, action: \.itemList) {
            ItemListFeature()
        }
        Reduce { state, action in
            switch action {
            case .itemList(.itemTapped(let item)):
                state.path.append(.detail(ItemDetailFeature.State(item: item)))
                return .none
            default:
                return .none
            }
        }
        .forEach(\.path, action: \.path) { Path() }
    }
}

// Root view
struct AppView: View {
    @Bindable var store: StoreOf<AppFeature>

    var body: some View {
        NavigationStack(path: $store.scope(state: \.path, action: \.path)) {
            ItemListView(store: store.scope(state: \.itemList, action: \.itemList))
        } destination: { store in
            switch store.case {
            case .detail(let store): ItemDetailView(store: store)
            }
        }
    }
}
```

---

## Effects

### fire-and-forget
```swift
return .run { _ in
    await analytics.track(.buttonTapped)
}
```

### Cancellable
```swift
// Cancel previous search when user types again
case .searchTextChanged(let query):
    state.query = query
    return .run { send in
        try await Task.sleep(for: .milliseconds(300))
        let results = try await searchService.search(query)
        await send(.resultsLoaded(results))
    }
    .cancellable(id: CancelID.search, cancelInFlight: true)

private enum CancelID { case search }
```

### Debounce (built-in)
```swift
return .run { send in
    let results = try await searchService.search(query)
    await send(.resultsLoaded(results))
}
.debounce(id: CancelID.search, for: .milliseconds(300), scheduler: DispatchQueue.main)
```

---

## Presentation (sheets, popovers)

Use `@Presents` for optional child features driven by presence.

```swift
@Reducer
struct ParentFeature {
    @ObservableState
    struct State {
        @Presents var detail: ItemDetailFeature.State?
    }

    enum Action {
        case showDetail(Item)
        case detail(PresentationAction<ItemDetailFeature.Action>)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .showDetail(let item):
                state.detail = ItemDetailFeature.State(item: item)
                return .none
            case .detail(.dismiss):
                state.detail = nil
                return .none
            case .detail:
                return .none
            }
        }
        .ifLet(\.$detail, action: \.detail) { ItemDetailFeature() }
    }
}

// View
.sheet(item: $store.scope(state: \.detail, action: \.detail)) { store in
    ItemDetailView(store: store)
}
```

---

## Testing

```swift
@Test
func loadItems() async {
    let items: [Item] = [.fixture()]

    let store = TestStore(initialState: ItemListFeature.State()) {
        ItemListFeature()
    } withDependencies: {
        $0.itemService.fetchItems = { items }
    }

    await store.send(.onAppear) {
        $0.isLoading = true
    }

    await store.receive(\.itemsLoaded.success) {
        $0.isLoading = false
        $0.items = items
    }
}

// Exhaustive mode off — test only what you care about
store.exhaustivity = .off

await store.send(.onAppear)
await store.receive(\.itemsLoaded.success) {
    $0.items = items
}
```

---

## When to Use TCA vs MVVM

| Scenario | TCA | MVVM |
|----------|-----|------|
| Complex state machines | ✅ | ⚠️ gets messy |
| Deep feature composition | ✅ | ⚠️ manual wiring |
| Exhaustive testing required | ✅ | ✅ (easier setup) |
| Small/medium features | ⚠️ boilerplate | ✅ |
| Team unfamiliar with TCA | ⚠️ steep curve | ✅ |
| Time travel debugging | ✅ | ❌ |
