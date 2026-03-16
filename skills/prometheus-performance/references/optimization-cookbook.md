# Optimization Cookbook

20+ specific optimization patterns with before/after metrics. Each pattern includes: scenario, measurement methodology, before metrics, implementation, after metrics.

---

## List & Scroll Performance

### 1. LazyVStack for Large Lists

**Scenario:** Screen displays a list of 1000+ items. Using `VStack` inside `ScrollView`.

**Before:**
```swift
ScrollView {
    VStack {
        ForEach(transactions) { tx in
            TransactionRow(transaction: tx)
        }
    }
}
// Initial render: 16s (all 1000 views created immediately)
// Memory: 800MB (all views in memory)
// Scroll FPS: N/A (can't even load)
```

**After:**
```swift
ScrollView {
    LazyVStack {
        ForEach(transactions) { tx in
            TransactionRow(transaction: tx)
        }
    }
}
// Initial render: 0.3s (only visible views created)
// Memory: 30MB (only visible + buffer views in memory)
// Scroll FPS: 60 FPS
```

**Measurement:** Time Profiler -> measure time from navigation start to first frame rendered.

---

### 2. Precomputed Cell Content

**Scenario:** Each list cell formats currency, dates, and status strings in its `body` property. SwiftUI recomputes `body` during scrolling.

**Before:**
```swift
struct TransactionRow: View {
    let transaction: Transaction

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(transaction.description)
                Text(transaction.date.formatted(.dateTime.month().day().year()))
            }
            Spacer()
            Text(transaction.amount.formatted(.currency(code: transaction.currencyCode)))
                .foregroundColor(transaction.amount >= 0 ? .green : .red)
        }
    }
}
// CPU per cell: 2ms (formatting runs on every body evaluation)
// During scroll: 120ms/frame (60 visible cells x 2ms = dropped frames)
```

**After:**
```swift
struct TransactionRowData: Equatable {
    let id: UUID
    let description: String
    let formattedDate: String
    let formattedAmount: String
    let amountColor: Color

    init(from transaction: Transaction) {
        self.id = transaction.id
        self.description = transaction.description
        self.formattedDate = transaction.date.formatted(.dateTime.month().day().year())
        self.formattedAmount = transaction.amount.formatted(.currency(code: transaction.currencyCode))
        self.amountColor = transaction.amount >= 0 ? .green : .red
    }
}

struct TransactionRow: View, Equatable {
    let data: TransactionRowData

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(data.description)
                Text(data.formattedDate)
            }
            Spacer()
            Text(data.formattedAmount)
                .foregroundColor(data.amountColor)
        }
    }
}

// In parent:
ForEach(rowData) { data in
    TransactionRow(data: data).equatable()
}
// CPU per cell: 0.1ms (no formatting, just reading precomputed strings)
// During scroll: 6ms/frame (60 x 0.1ms = smooth 60 FPS)
```

**Measurement:** Time Profiler -> focus on main thread during scroll -> check self time in body getters.

---

### 3. Equatable View Optimization

**Scenario:** A parent view's state changes (e.g., selected tab) causing all child views to re-evaluate their `body`, even though their data hasn't changed.

**Before:**
```swift
struct DashboardView: View {
    @State private var selectedTab = 0
    @StateObject private var viewModel = DashboardViewModel()

    var body: some View {
        VStack {
            Picker("Tab", selection: $selectedTab) { /* ... */ }

            // ALL of these re-render when selectedTab changes
            BalanceCard(balance: viewModel.balance)
            TransactionList(items: viewModel.transactions)
            QuickActions(actions: viewModel.actions)
        }
    }
}
// Tab switch: 50ms (all children re-render even though their data didn't change)
```

**After:**
```swift
struct BalanceCard: View, Equatable {
    let balance: Decimal

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.balance == rhs.balance
    }

    var body: some View {
        // Only called when balance actually changes
        Text(balance.formatted(.currency(code: "USD")))
    }
}

// In parent:
BalanceCard(balance: viewModel.balance).equatable()
TransactionList(items: viewModel.transactions).equatable()
QuickActions(actions: viewModel.actions).equatable()
// Tab switch: 2ms (children skip re-render because their inputs haven't changed)
```

**Measurement:** Add `let _ = Self._printChanges()` to each view's body to see which views re-render and why.

---

## Image Performance

### 4. Image Downsampling for Thumbnails

**Scenario:** Displaying user avatars or thumbnails that load full-resolution images.

**Before:**
```swift
// Loading a 4000x3000 photo (12MB) for a 44x44pt avatar
Image(uiImage: UIImage(data: imageData)!)
    .resizable()
    .frame(width: 44, height: 44)
// Memory per image: 48MB (4000 x 3000 x 4 bytes = uncompressed bitmap)
// Decode time: 200ms per image
// 20 avatars on screen: 960MB memory (crash on older devices)
```

**After:**
```swift
func downsample(data: Data, to pointSize: CGSize, scale: CGFloat = UIScreen.main.scale) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithData(data as CFData, imageSourceOptions) else {
        return nil
    }

    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary

    guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
        return nil
    }

    return UIImage(cgImage: downsampledImage)
}

// Usage
let thumbnail = downsample(data: imageData, to: CGSize(width: 44, height: 44))
Image(uiImage: thumbnail!)
    .frame(width: 44, height: 44)
// Memory per image: 0.03MB (44 x 44 x 4 bytes x 9 scale = tiny)
// Decode time: 5ms per image
// 20 avatars on screen: 0.6MB memory
```

**Measurement:** Allocations instrument -> filter by `CGImage` -> compare persistent bytes.

---

### 5. Async Image Loading with Cache

**Scenario:** Images loaded synchronously on the main thread during cell configuration.

**Before:**
```swift
struct ProductRow: View {
    let product: Product

    var body: some View {
        HStack {
            // Synchronous load on main thread = UI freeze
            if let data = try? Data(contentsOf: product.imageURL),
               let image = UIImage(data: data) {
                Image(uiImage: image)
                    .frame(width: 60, height: 60)
            }
            Text(product.name)
        }
    }
}
// Scroll FPS: 15 FPS (each cell blocks main thread for 100-500ms)
```

**After:**
```swift
struct ProductRow: View {
    let product: Product

    var body: some View {
        HStack {
            AsyncImage(url: product.imageURL) { phase in
                switch phase {
                case .success(let image):
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fill)
                case .failure:
                    Image(systemName: "photo")
                        .foregroundColor(.secondary)
                case .empty:
                    ProgressView()
                @unknown default:
                    EmptyView()
                }
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            Text(product.name)
        }
    }
}
// Scroll FPS: 60 FPS (images load asynchronously, placeholder shown)
// Network: URLCache handles caching automatically
```

**Measurement:** Time Profiler -> main thread -> look for Data(contentsOf:) or URLSession on Thread 1.

---

## Network Performance

### 6. Parallel Network Requests with async let

**Scenario:** Dashboard screen makes 5 sequential API calls before displaying.

**Before:**
```swift
func loadDashboard() async throws -> Dashboard {
    let user = try await api.fetchUser()           // 500ms
    let balance = try await api.fetchBalance()      // 300ms
    let transactions = try await api.fetchRecent()  // 400ms
    let notifications = try await api.fetchNotifs()  // 200ms
    let settings = try await api.fetchSettings()    // 100ms

    return Dashboard(user: user, balance: balance,
                    transactions: transactions,
                    notifications: notifications,
                    settings: settings)
}
// Total: 1500ms (sum of all calls, sequential)
```

**After:**
```swift
func loadDashboard() async throws -> Dashboard {
    async let user = api.fetchUser()
    async let balance = api.fetchBalance()
    async let transactions = api.fetchRecent()
    async let notifications = api.fetchNotifs()
    async let settings = api.fetchSettings()

    return try await Dashboard(
        user: user,
        balance: balance,
        transactions: transactions,
        notifications: notifications,
        settings: settings
    )
}
// Total: 500ms (bounded by slowest call, parallel)
// Improvement: 67% faster
```

**Measurement:** Network instrument -> compare request waterfall (sequential vs parallel bars).

---

### 7. Conditional Requests (ETags)

**Scenario:** App fetches the same data repeatedly even when it hasn't changed.

**Before:**
```swift
// Always fetches full response
func fetchTransactions() async throws -> [Transaction] {
    let (data, _) = try await URLSession.shared.data(from: transactionsURL)
    return try JSONDecoder().decode([Transaction].self, from: data)
}
// Every call: full response downloaded (50KB), decoded, rendered
```

**After:**
```swift
actor TransactionCache {
    private var etag: String?
    private var cached: [Transaction]?

    func fetch() async throws -> [Transaction] {
        var request = URLRequest(url: transactionsURL)
        if let etag = etag {
            request.setValue(etag, forHTTPHeaderField: "If-None-Match")
        }

        let (data, response) = try await URLSession.shared.data(for: request)
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        if httpResponse.statusCode == 304 {
            // Not modified -- return cached data
            guard let cached = cached else {
                throw NetworkError.cacheEmpty
            }
            return cached
        }

        // New data -- cache it
        let transactions = try JSONDecoder().decode([Transaction].self, from: data)
        self.etag = httpResponse.value(forHTTPHeaderField: "ETag")
        self.cached = transactions
        return transactions
    }
}
// When unchanged: 304 response (200 bytes), no decode, no render update
// Saves: 50KB bandwidth + 20ms decode time per unchanged request
```

**Measurement:** Network instrument -> compare response sizes and counts.

---

### 8. Request Deduplication

**Scenario:** Multiple views request the same data simultaneously (e.g., balance shown in header and transaction list).

**Before:**
```swift
// Two views independently fetch the same data
// HeaderView.task { balance = try await api.fetchBalance() }
// TransactionView.task { balance = try await api.fetchBalance() }
// Result: 2 identical API calls
```

**After:**
```swift
actor RequestDeduplicator {
    private var inFlightRequests: [String: Task<Any, Error>] = [:]

    func deduplicated<T: Sendable>(key: String, work: @Sendable () async throws -> T) async throws -> T {
        if let existing = inFlightRequests[key] {
            return try await existing.value as! T
        }

        let task = Task<Any, Error> {
            let result = try await work()
            inFlightRequests[key] = nil
            return result
        }

        inFlightRequests[key] = task
        return try await task.value as! T
    }
}

// Usage
let balance = try await deduplicator.deduplicated(key: "balance") {
    try await api.fetchBalance()
}
// Result: 1 API call shared between both views
```

**Measurement:** Network instrument -> count duplicate requests to same endpoint.

---

## Memory Performance

### 9. Bounded Image Cache

**Scenario:** App caches downloaded images in a dictionary that grows unboundedly.

**Before:**
```swift
// Unbounded cache
var imageCache: [URL: UIImage] = [:]

func loadImage(from url: URL) async -> UIImage? {
    if let cached = imageCache[url] { return cached }
    let (data, _) = try? await URLSession.shared.data(from: url)
    let image = data.flatMap(UIImage.init)
    imageCache[url] = image // Never evicted!
    return image
}
// After browsing 100 images: 500MB+ cache, eventually OOM
```

**After:**
```swift
// NSCache: automatically evicts under memory pressure, thread-safe
let imageCache = NSCache<NSURL, UIImage>()

func configure() {
    imageCache.countLimit = 100        // Max 100 images
    imageCache.totalCostLimit = 50_000_000  // Max 50MB
}

func loadImage(from url: URL) async -> UIImage? {
    if let cached = imageCache.object(forKey: url as NSURL) { return cached }
    guard let (data, _) = try? await URLSession.shared.data(from: url),
          let image = UIImage(data: data) else { return nil }

    let cost = data.count
    imageCache.setObject(image, forKey: url as NSURL, cost: cost)
    return image
}
// Memory bounded to 50MB regardless of browsing history
// Automatic eviction under memory pressure
```

**Measurement:** Allocations instrument -> Mark Generation before and after browsing -> check growth.

---

### 10. Autoreleasepool for Batch Processing

**Scenario:** Processing a large batch of items in a loop causes memory spike.

**Before:**
```swift
func processAllImages(_ urls: [URL]) async {
    for url in urls {
        let data = try? Data(contentsOf: url)
        let image = data.flatMap(UIImage.init)
        let processed = applyFilter(image)
        save(processed)
        // Temporary objects accumulate until the loop ends
    }
}
// Memory during processing: 2GB peak (all temporaries alive simultaneously)
```

**After:**
```swift
func processAllImages(_ urls: [URL]) async {
    for url in urls {
        autoreleasepool {
            let data = try? Data(contentsOf: url)
            let image = data.flatMap(UIImage.init)
            let processed = applyFilter(image)
            save(processed)
            // Temporaries freed at end of each iteration
        }
    }
}
// Memory during processing: 50MB peak (only current iteration's temporaries)
```

**Measurement:** Allocations instrument -> watch live memory graph during batch processing.

---

## Launch Performance

### 11. Deferred Initialization

**Scenario:** `didFinishLaunchingWithOptions` initializes everything synchronously.

**Before:**
```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    AnalyticsSDK.configure(key: analyticsKey)        // 200ms
    CrashReporter.start()                             // 100ms
    FeatureFlags.sync()                               // 300ms (network!)
    PushNotifications.register()                       // 50ms
    DatabaseMigration.runPending()                     // 500ms
    PreloadCache.warmUp()                              // 200ms
    return true
}
// Total didFinishLaunching: 1350ms (user stares at launch screen)
```

**After:**
```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // CRITICAL PATH ONLY: minimum needed to show first screen
    CrashReporter.start()  // 100ms -- need this first for crash reports

    // Everything else deferred
    Task.detached(priority: .background) {
        await AnalyticsSDK.configure(key: analyticsKey)
        await FeatureFlags.sync()
        await PreloadCache.warmUp()
    }

    Task.detached(priority: .utility) {
        await DatabaseMigration.runPending()
    }

    // Register push after a delay (doesn't affect initial render)
    DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
        PushNotifications.register()
    }

    return true
}
// Total didFinishLaunching: 100ms (CrashReporter only)
// First screen visible: 200ms (vs 1350ms before)
// Everything else loads in background
```

**Measurement:** App Launch instrument -> compare post-main time.

---

### 12. Skeleton UI While Loading

**Scenario:** Screen shows a blank/loading spinner while data loads, then pops in all at once.

**Before:**
```swift
struct AccountView: View {
    @StateObject private var vm = AccountViewModel()

    var body: some View {
        Group {
            if vm.isLoading {
                ProgressView() // User sees spinner for 1-2 seconds
            } else {
                AccountContent(data: vm.accountData)
            }
        }
        .task { await vm.load() }
    }
}
// Perceived load time: 1.5s (spinner -> content pop-in)
```

**After:**
```swift
struct AccountView: View {
    @StateObject private var vm = AccountViewModel()

    var body: some View {
        AccountContent(data: vm.accountData)
            .redacted(reason: vm.isLoading ? .placeholder : [])
            .task { await vm.load() }
    }
}

// AccountContent uses the same layout for loading and loaded states
// SwiftUI's .redacted(reason: .placeholder) shows shimmer effect
// Perceived load time: 0.3s (instant skeleton, smooth transition to content)
```

**Measurement:** Time to interactive (signpost from task start to isLoading = false).

---

## Core Data Performance

### 13. Background Context Fetches

**Scenario:** Core Data fetch request blocks the main thread during list display.

**Before:**
```swift
@MainActor
func loadTransactions() {
    let request = Transaction.fetchRequest()
    request.sortDescriptors = [NSSortDescriptor(keyPath: \Transaction.date, ascending: false)]
    request.fetchLimit = 100

    // Blocks main thread during fetch
    let results = try? viewContext.fetch(request)
    self.transactions = results ?? []
}
// Main thread blocked: 150ms for 10,000 records
// UI freezes during fetch
```

**After:**
```swift
@MainActor
func loadTransactions() async {
    let results = await withCheckedContinuation { continuation in
        let backgroundContext = persistentContainer.newBackgroundContext()
        backgroundContext.perform {
            let request = Transaction.fetchRequest()
            request.sortDescriptors = [NSSortDescriptor(keyPath: \Transaction.date, ascending: false)]
            request.fetchLimit = 100
            request.returnsObjectsAsFaults = false // Prefetch to avoid faulting on main thread

            let results = (try? backgroundContext.fetch(request)) ?? []
            let objectIDs = results.map(\.objectID)
            continuation.resume(returning: objectIDs)
        }
    }

    // Fetch on main context using IDs (very fast, objects likely in cache)
    self.transactions = results.compactMap { viewContext.object(with: $0) as? Transaction }
}
// Main thread blocked: 5ms (just materializing from cache)
// Background fetch: 150ms (but doesn't block UI)
```

**Measurement:** Time Profiler -> main thread -> look for `NSSQLiteStatement` or `executeFetchRequest`.

---

### 14. Batch Inserts

**Scenario:** Importing 10,000 records from an API response into Core Data.

**Before:**
```swift
func importTransactions(_ items: [TransactionDTO]) async {
    let context = viewContext
    for item in items {
        let entity = Transaction(context: context)
        entity.id = item.id
        entity.amount = item.amount
        entity.date = item.date
        // ... set all properties
    }
    try? context.save() // One massive save
}
// Time: 45 seconds for 10,000 records
// Memory: 500MB peak (all managed objects in memory)
```

**After:**
```swift
func importTransactions(_ items: [TransactionDTO]) async {
    let backgroundContext = persistentContainer.newBackgroundContext()
    backgroundContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy

    await backgroundContext.perform {
        let batchSize = 500
        for batch in items.chunked(into: batchSize) {
            autoreleasepool {
                for item in batch {
                    let entity = Transaction(context: backgroundContext)
                    entity.id = item.id
                    entity.amount = item.amount
                    entity.date = item.date
                }
                try? backgroundContext.save()
                backgroundContext.reset() // Release managed objects from memory
            }
        }
    }
}
// Time: 3 seconds for 10,000 records
// Memory: 30MB peak (only current batch in memory)
```

**Measurement:** Allocations instrument during import -> check peak memory and persistent growth.

---

## SwiftUI-Specific Optimizations

### 15. @StateObject vs @ObservedObject

**Scenario:** Child view receives a ViewModel but its parent re-renders frequently, causing the ViewModel to be recreated.

**Before:**
```swift
struct ParentView: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            Button("Increment") { counter += 1 }
            // ObservedObject: ViewModel RECREATED on every parent re-render
            ChildView(viewModel: ChildViewModel())
        }
    }
}

struct ChildView: View {
    @ObservedObject var viewModel: ChildViewModel
    // viewModel.loadData() called in .task, but task restarts on recreation
}
// Every parent re-render: ChildViewModel recreated, data refetched
```

**After:**
```swift
struct ParentView: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            Button("Increment") { counter += 1 }
            ChildView() // Child manages its own ViewModel
        }
    }
}

struct ChildView: View {
    @StateObject private var viewModel = ChildViewModel()
    // StateObject: persists across parent re-renders
}
// Parent re-render: ChildViewModel persists, no data refetch
```

**Measurement:** Add `init()` logging to ViewModel to count instantiations.

---

### 16. Minimizing View Identity Changes

**Scenario:** Using `id()` modifier or conditional views that cause SwiftUI to destroy and recreate views.

**Before:**
```swift
// This destroys and recreates the entire list on every data update
List(items) { item in
    ItemRow(item: item)
}
.id(UUID()) // FORCES complete view recreation!
// Result: scroll position lost, animations broken, performance degraded
```

**After:**
```swift
// Let SwiftUI's diffing handle updates naturally
List(items) { item in
    ItemRow(item: item)
}
// SwiftUI compares item.id values and only updates changed rows
// Scroll position preserved, smooth animations, optimal performance
```

---

### 17. Reducing View Hierarchy Depth

**Scenario:** Deeply nested container views cause slow layout computation.

**Before:**
```swift
VStack {
    HStack {
        VStack {
            HStack {
                VStack {
                    // 5+ levels deep
                    Text("Hello")
                }
            }
        }
    }
}
// Layout computation: O(depth * children)
```

**After:**
```swift
// Flatten where possible, extract components
HStack {
    Text("Hello")
    // Use custom Layout for complex arrangements instead of nesting
}
// Or use a custom Layout protocol implementation for complex layouts
```

**Measurement:** Xcode View Hierarchy Debugger -> check view depth.

---

## General Patterns

### 18. Debouncing Search Input

**Scenario:** Search query triggers API call on every keystroke.

**Before:**
```swift
struct SearchView: View {
    @State private var query = ""

    var body: some View {
        TextField("Search", text: $query)
            .onChange(of: query) { _, newValue in
                Task { await search(newValue) } // API call on EVERY keystroke
            }
    }
}
// Typing "hello": 5 API calls (h, he, hel, hell, hello)
// Server load: 5x necessary
// UI: results flicker as each response arrives
```

**After:**
```swift
struct SearchView: View {
    @State private var query = ""
    @State private var debouncedQuery = ""

    var body: some View {
        TextField("Search", text: $query)
            .onChange(of: query) { _, newValue in
                // Debounce: wait 300ms after last keystroke
            }
            .task(id: query) {
                try? await Task.sleep(for: .milliseconds(300))
                guard !Task.isCancelled else { return }
                await search(query)
            }
    }
}
// Typing "hello" (normal speed): 1 API call ("hello")
// Server load: 1x
// UI: single clean result
```

**Measurement:** Network instrument -> count search API calls while typing.

---

### 19. Avoiding Redundant State Updates

**Scenario:** ViewModel publishes state changes even when the value hasn't actually changed, causing unnecessary view re-renders.

**Before:**
```swift
@MainActor
final class DashboardViewModel: ObservableObject {
    @Published var balance: Decimal = 0

    func refresh() async {
        let newBalance = try? await api.fetchBalance()
        balance = newBalance ?? balance // Publishes even if same value!
    }
}
// Every refresh: view re-renders even if balance hasn't changed
```

**After:**
```swift
@MainActor
final class DashboardViewModel: ObservableObject {
    @Published var balance: Decimal = 0

    func refresh() async {
        let newBalance = try? await api.fetchBalance()
        if let newBalance, newBalance != balance {
            balance = newBalance // Only publish when actually changed
        }
    }
}
// Refresh with same balance: no re-render
// Refresh with new balance: single re-render
```

**Measurement:** Add `let _ = Self._printChanges()` to the view's body.

---

### 20. String Interpolation vs Concatenation

**Scenario:** Building complex strings in a tight loop.

**Before:**
```swift
func buildReport(transactions: [Transaction]) -> String {
    var result = ""
    for tx in transactions {
        result += "Date: " + tx.date.description + " Amount: " + tx.amount.description + "\n"
        // String concatenation: O(n^2) for n transactions (copies entire string each time)
    }
    return result
}
// 10,000 transactions: 2.5 seconds
```

**After:**
```swift
func buildReport(transactions: [Transaction]) -> String {
    var parts: [String] = []
    parts.reserveCapacity(transactions.count)
    for tx in transactions {
        parts.append("Date: \(tx.date) Amount: \(tx.amount)")
    }
    return parts.joined(separator: "\n")
    // Array append: O(1) amortized, joined: O(n)
}
// 10,000 transactions: 0.05 seconds (50x faster)
```

**Measurement:** Time Profiler -> self time in string operations.
