# SwiftUI Views Reference

Architecture-agnostic. Covers view composition, layout, lifecycle, and best practices for iOS 17+ / macOS 14+.

---

## View Composition

### Small, Focused Views
Break complex views into small composable pieces. Extract when a subview has a clear responsibility or is reused.

```swift
struct UserCard: View {
    var body: some View {
        HStack {
            AvatarView(url: user.avatarURL)
            UserInfoView(user: user)
            Spacer()
            BadgeView(count: user.unreadCount)
        }
        .padding()
        .background(.regularMaterial)
        .clipShape(.rect(cornerRadius: 12))
    }
}

// Each subview is small and testable in isolation
private struct AvatarView: View {
    let url: URL?
    var body: some View { /* ... */ }
}
```

### Computed Vars Over Body Clutter
Extract subviews as `private var` when they don't need parameters — avoids unnecessary type declarations.

```swift
struct ProfileView: View {
    let profile: Profile

    var body: some View {
        VStack {
            header
            Divider()
            statsGrid
        }
    }

    private var header: some View {
        HStack {
            Text(profile.name).font(.title)
            Spacer()
            Text(profile.role).foregroundStyle(.secondary)
        }
    }

    private var statsGrid: some View {
        LazyVGrid(columns: [.init(.flexible()), .init(.flexible())]) {
            ForEach(profile.stats) { StatCell(stat: $0) }
        }
    }
}
```

### ViewBuilder for Conditional Composition
Use `@ViewBuilder` when a helper needs to return different view types.

```swift
@ViewBuilder
private func badge(for status: Status) -> some View {
    switch status {
    case .active:   Label("Active", systemImage: "checkmark.circle.fill").foregroundStyle(.green)
    case .pending:  Label("Pending", systemImage: "clock").foregroundStyle(.orange)
    case .inactive: EmptyView()
    }
}
```

---

## Layout

### Stack Fundamentals
- `VStack` / `HStack` — static spacing, lazy alternatives for long lists
- `ZStack` — layering; use `alignment:` to pin anchors
- `Grid` — two-dimensional alignment (iOS 16+), great for forms and grids with aligned columns

```swift
// Grid aligns columns across rows — LazyVGrid does not
Grid(alignment: .leading, horizontalSpacing: 16, verticalSpacing: 8) {
    ForEach(items) { item in
        GridRow {
            Text(item.label).foregroundStyle(.secondary)
            Text(item.value).bold()
        }
    }
}
```

### Spacer and Padding
```swift
// Push content apart
HStack {
    Text("Left")
    Spacer()
    Text("Right")
}

// Consistent insets — prefer semantic over specific values
.padding()            // all edges, system default
.padding(.horizontal) // left + right only
.padding(.top, 24)    // specific edge + value
```

### Frame
```swift
// Fixed size
.frame(width: 44, height: 44)

// Flexible with constraints
.frame(maxWidth: .infinity, alignment: .leading)
.frame(minHeight: 200, maxHeight: 400)

// Aspect ratio
.aspectRatio(16/9, contentMode: .fit)
```

### Alignment Guides (advanced)
Use when built-in alignment doesn't match your design.

```swift
HStack(alignment: .midIcon) {
    Image(systemName: "star.fill")
        .alignmentGuide(.midIcon) { $0[VerticalAlignment.center] }
    VStack(alignment: .leading) {
        Text("Title").font(.headline)
        Text("Subtitle").font(.caption)
            .alignmentGuide(.midIcon) { $0[VerticalAlignment.center] }
    }
}

extension VerticalAlignment {
    private enum MidIcon: AlignmentID {
        static func defaultValue(in d: ViewDimensions) -> CGFloat { d[.top] }
    }
    static let midIcon = VerticalAlignment(MidIcon.self)
}
```

---

## Modifiers

### Order Matters
Modifiers are applied inside-out. Background and padding order changes visual result.

```swift
// Padding OUTSIDE background → background hugs content
Text("Hello")
    .padding()
    .background(.blue)

// Background OUTSIDE padding → background includes padding
Text("Hello")
    .background(.blue)
    .padding()
```

### Custom Modifiers
Group related modifiers that are reused across views.

```swift
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(.rect(cornerRadius: 12))
            .shadow(color: .black.opacity(0.08), radius: 4, y: 2)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardStyle())
    }
}

// Usage
Text("Hello").cardStyle()
```

### Conditional Modifiers
Avoid `if` branching on modifiers — use `.opacity`, `.hidden()`, or helper extensions.

```swift
// Prefer
Text("Value")
    .opacity(isEnabled ? 1 : 0.4)
    .allowsHitTesting(isEnabled)

// Extension for clean conditional application
extension View {
    @ViewBuilder
    func `if`<Content: View>(_ condition: Bool, transform: (Self) -> Content) -> some View {
        if condition { transform(self) } else { self }
    }
}

Text("Value").if(isPro) { $0.overlay(ProBadge(), alignment: .topTrailing) }
```

---

## Lists and Scroll Views

### List
```swift
List {
    Section("Recent") {
        ForEach(items) { item in
            ItemRow(item: item)
                .swipeActions(edge: .trailing) {
                    Button(role: .destructive) { delete(item) } label: {
                        Label("Delete", systemImage: "trash")
                    }
                }
        }
    }
}
.listStyle(.insetGrouped)
```

### LazyVStack in ScrollView (preferred for complex layouts)
Gives more layout control than `List` while still being lazy.

```swift
ScrollView {
    LazyVStack(spacing: 16, pinnedViews: .sectionHeaders) {
        Section {
            ForEach(items) { ItemCard(item: $0) }
        } header: {
            SectionHeader(title: "Items")
                .background(.bar) // sticky header needs background
        }
    }
    .padding(.horizontal)
}
```

### ScrollViewReader (programmatic scrolling)
```swift
ScrollViewReader { proxy in
    ScrollView {
        ForEach(messages) { msg in
            MessageBubble(message: msg).id(msg.id)
        }
    }
    .onChange(of: messages.count) {
        withAnimation { proxy.scrollTo(messages.last?.id, anchor: .bottom) }
    }
}
```

---

## View Lifecycle

### task(_:) — Async Work
Preferred over `onAppear` for async operations. Cancels automatically when view disappears.

```swift
.task {
    await viewModel.loadData()
}

// With dependencies — re-runs when value changes
.task(id: selectedFilter) {
    await viewModel.load(filter: selectedFilter)
}
```

### onAppear / onDisappear
Use for synchronous side effects only (analytics, scroll state, etc.).

```swift
.onAppear { analytics.track(.screenView("Detail")) }
.onDisappear { player.pause() }
```

### onChange
```swift
// iOS 17+ signature — new value in closure
.onChange(of: searchText) { _, newValue in
    performSearch(newValue)
}

// Or without parameters if you don't need value
.onChange(of: isPresented) {
    if isPresented { prepareContent() }
}
```

---

## Geometry and Size Reading

### GeometryReader (use sparingly)
Reads available space — causes layout passes. Prefer `.containerRelativeFrame` when possible.

```swift
// Modern alternative (iOS 17+)
Color.blue
    .containerRelativeFrame(.horizontal) { size, _ in size * 0.8 }

// Classic GeometryReader when you truly need measured size
GeometryReader { proxy in
    Circle()
        .frame(width: proxy.size.width)
}
```

### onGeometryChange (iOS 18+)
Read size without full GeometryReader overhead.

```swift
@State private var headerHeight: CGFloat = 0

Text("Header")
    .onGeometryChange(for: CGFloat.self) { proxy in
        proxy.size.height
    } action: { height in
        headerHeight = height
    }
```

---

## Sheets, Alerts, and Overlays

### Sheet and fullScreenCover
```swift
.sheet(isPresented: $showDetail) {
    DetailView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
}

// Item-based (preferred when passing data)
.sheet(item: $selectedItem) { item in
    ItemDetailView(item: item)
}
```

### Alerts
```swift
.alert("Delete Item?", isPresented: $showAlert) {
    Button("Delete", role: .destructive) { confirmDelete() }
    Button("Cancel", role: .cancel) { }
} message: {
    Text("This action cannot be undone.")
}

// Error alert pattern
.alert("Error", isPresented: .constant(error != nil), presenting: error) { _ in
    Button("OK") { error = nil }
} message: { err in
    Text(err.localizedDescription)
}
```

### Overlay vs ZStack
Prefer `.overlay` — it doesn't affect the layout of sibling views.

```swift
// Good — badge floats over without affecting layout
Image(systemName: "bell")
    .overlay(alignment: .topTrailing) {
        Circle().fill(.red).frame(width: 8, height: 8)
            .opacity(hasNotifications ? 1 : 0)
    }
```

---

## Performance

### equatable() — Skip Redundant Redraws
For expensive views with value-type inputs that conform to `Equatable`.

```swift
struct HeavyChart: View, Equatable {
    let data: [DataPoint] // must be Equatable

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.data == rhs.data
    }

    var body: some View { /* expensive rendering */ }
}

// Used as:
HeavyChart(data: data).equatable()
```

### drawingGroup() — Flatten Composite Views
Rasterizes a complex view hierarchy into a single Metal layer. Use for views with many overlapping elements.

```swift
// Good for particle effects, confetti, dense icon grids
ParticleCanvas()
    .drawingGroup()
```

### Avoid Identifiable Anti-Patterns
```swift
// Bad — recomputes and recreates all rows on any change
ForEach(items.indices, id: \.self) { i in ItemRow(item: items[i]) }

// Good — stable identity, SwiftUI can diff efficiently
ForEach(items) { item in ItemRow(item: item) } // Item: Identifiable
```

---

## Common Patterns

### Empty State
```swift
struct EmptyStateView: View {
    let title: String
    let subtitle: String
    let systemImage: String
    var action: (() -> Void)? = nil
    var actionLabel: String = "Try Again"

    var body: some View {
        ContentUnavailableView {
            Label(title, systemImage: systemImage)
        } description: {
            Text(subtitle)
        } actions: {
            if let action {
                Button(actionLabel, action: action)
                    .buttonStyle(.borderedProminent)
            }
        }
    }
}
```

### Loading State
```swift
// Overlay shimmer/progress without replacing content
.overlay {
    if isLoading {
        ZStack {
            Rectangle().fill(.background.opacity(0.6))
            ProgressView()
        }
    }
}

// Or use redacted for skeleton screens
List(placeholderItems) { item in
    ItemRow(item: item)
}
.redacted(reason: isLoading ? .placeholder : [])
.shimmer(isActive: isLoading) // custom modifier
```

### Pull to Refresh
```swift
List(items) { item in ItemRow(item: item) }
    .refreshable {
        await viewModel.refresh() // suspends until done
    }
```

### Searchable
```swift
@State private var searchText = ""

NavigationStack {
    List(filteredItems) { item in ItemRow(item: item) }
        .searchable(text: $searchText, placement: .navigationBarDrawer)
}

private var filteredItems: [Item] {
    searchText.isEmpty ? items : items.filter { $0.matches(searchText) }
}
```

### Animating views

- Strongly prefer to use the `@Animatable` macro over creating `animatableData` manually – the macro automatically adds conformance to the `Animatable` protocol and creates the correct `animatableData` property. If some properties should not or cannot be animated (e.g. Booleans, integers, etc), mark them `@AnimatableIgnored`.
- Never use `animation(_ animation: Animation?)`; always provide a value to watch, such as `.animation(.bouncy, value: score)`.
- Chaining animations must be done using a `completion` closure passed to `withAnimation()`, rather than trying to execute multiple `withAnimation()` calls using delays.
