# MVVM Architecture Reference

SwiftUI-native MVVM using `@Observable` (iOS 17+ / macOS 14+). No Combine required.

---

## Core Pattern

```
View  ──reads──▶  ViewModel (@Observable)  ──calls──▶  Service / Repository
  └──sends actions via methods                              └──returns data (async)
```

- **View**: renders state, forwards user actions to ViewModel
- **ViewModel**: holds state, runs async work, owns business logic
- **Service/Repository**: pure data access, no UI knowledge

---

## ViewModel

### Base Structure

```swift
@Observable
final class ItemListViewModel {
    // MARK: - State (read by View)
    private(set) var items: [Item] = []
    private(set) var isLoading = false
    private(set) var error: AppError?

    // MARK: - Dependencies
    private let service: ItemServiceProtocol

    init(service: ItemServiceProtocol = ItemService()) {
        self.service = service
    }

    // MARK: - Intent Methods
    func load() async {
        isLoading = true
        defer { isLoading = false }
        do {
            items = try await service.fetchItems()
        } catch {
            self.error = AppError(error)
        }
    }

    func delete(_ item: Item) async {
        do {
            try await service.delete(item)
            items.removeAll { $0.id == item.id }
        } catch {
            self.error = AppError(error)
        }
    }
}
```

### Rules
- Mark all state `private(set)` — Views never mutate ViewModel state directly
- One ViewModel per screen, not per component
- Dependencies injected via `init` — never instantiated inside ViewModel
- No `import SwiftUI` in ViewModel — it's a business logic layer

---

## View

```swift
struct ItemListView: View {
    @State private var viewModel = ItemListViewModel()

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
                .swipeActions {
                    Button(role: .destructive) {
                        Task { await viewModel.delete(item) }
                    } label: {
                        Label("Delete", systemImage: "trash")
                    }
                }
        }
        .overlay { if viewModel.isLoading { ProgressView() } }
        .alert("Error", isPresented: .constant(viewModel.error != nil), presenting: viewModel.error) { _ in
            Button("OK") { viewModel.error = nil }
        } message: { Text($0.localizedDescription) }
        .task { await viewModel.load() }
        .navigationTitle("Items")
    }
}
```

### Rules
- `@State private var viewModel` — view owns and creates its ViewModel
- Never pass a ViewModel as a parameter — pass the data it needs instead
- Child views receive plain values, not the ViewModel
- Use `Task { }` inside synchronous callbacks (button actions, swipe actions)

---

## Passing Data to Child Views

```swift
// ✅ Pass values — child is independent and reusable
ItemRow(item: item)

// ❌ Never pass the parent ViewModel — creates tight coupling
ItemRow(viewModel: viewModel)
```

If a child needs to trigger actions on the parent, pass closures:

```swift
ItemRow(item: item, onDelete: { Task { await viewModel.delete(item) } })
```

---

## Child ViewModels (detail screens)

Create a new ViewModel for each screen pushed onto the navigation stack.

```swift
struct ItemDetailView: View {
    let itemID: Item.ID

    @State private var viewModel: ItemDetailViewModel?

    var body: some View {
        Group {
            if let viewModel {
                ItemDetailContent(viewModel: viewModel)
            } else {
                ProgressView()
            }
        }
        .task {
            viewModel = ItemDetailViewModel(itemID: itemID)
            await viewModel?.load()
        }
    }
}
```

---

## Environment Injection (shared dependencies)

For services needed across many screens (auth, database, etc.):

```swift
// Define key
private struct ItemServiceKey: EnvironmentKey {
    static let defaultValue: ItemServiceProtocol = ItemService()
}

extension EnvironmentValues {
    var itemService: ItemServiceProtocol {
        get { self[ItemServiceKey.self] }
        set { self[ItemServiceKey.self] = newValue }
    }
}

// Inject at root
ContentView()
    .environment(\.itemService, ItemService(config: .production))

// Consume in ViewModel init
@Observable
final class ItemListViewModel {
    init(service: ItemServiceProtocol) { self.service = service }
}

// Read in View, pass to ViewModel
struct ItemListView: View {
    @Environment(\.itemService) private var itemService
    @State private var viewModel: ItemListViewModel?

    var body: some View {
        // ...
        .task { viewModel = ItemListViewModel(service: itemService) }
    }
}
```

---

## Shared State (cross-screen)

For state that must be shared across unrelated screens, use an `@Observable` store injected via Environment:

```swift
@Observable
final class AppStore {
    var currentUser: User?
    var cart: Cart = Cart()
}

// Inject at root
@main
struct MyApp: App {
    @State private var store = AppStore()

    var body: some Scene {
        WindowGroup {
            ContentView().environment(store)
        }
    }
}

// Consume anywhere in the tree
struct ProfileView: View {
    @Environment(AppStore.self) private var store

    var body: some View {
        Text(store.currentUser?.name ?? "Guest")
    }
}
```

---

## Testing

```swift
// Protocol-based service for easy mocking
protocol ItemServiceProtocol {
    func fetchItems() async throws -> [Item]
    func delete(_ item: Item) async throws
}

final class MockItemService: ItemServiceProtocol {
    var stubbedItems: [Item] = []
    var deleteCalledWith: Item?

    func fetchItems() async throws -> [Item] { stubbedItems }
    func delete(_ item: Item) async throws { deleteCalledWith = item }
}

// ViewModel test
@Test func loadsItems() async {
    let mock = MockItemService()
    mock.stubbedItems = [.fixture()]
    let vm = ItemListViewModel(service: mock)

    await vm.load()

    #expect(vm.items.count == 1)
    #expect(!vm.isLoading)
}
```

---

## Navigation

Coordinator-free — use `NavigationStack` with a path array in the ViewModel or a dedicated `NavigationStore`.

```swift
@Observable
final class AppNavigationStore {
    var path = NavigationPath()

    func push(_ destination: AppDestination) {
        path.append(destination)
    }

    func popToRoot() {
        path.removeLast(path.count)
    }
}

// Root
NavigationStack(path: $navStore.path) {
    ItemListView()
        .navigationDestination(for: AppDestination.self) { destination in
            switch destination {
            case .detail(let id): ItemDetailView(itemID: id)
            case .settings:       SettingsView()
            }
        }
}
```
