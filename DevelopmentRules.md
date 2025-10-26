# Development Rules (v3.6)

Comprehensive development standards for AllMyStuff.

## Overview

This document defines the development standards, architectural patterns, and coding conventions for the AllMyStuff project. It covers everything from core principles to detailed code organization rules.

> Important: This document is the single source of truth for AllMyStuff development standards.

> Note: This reference document exceeds 1500 lines. Documentation files serving as comprehensive standards may exceed code file length limits. The 1500-line guideline (§21, Rule 20) applies to implementation files where complexity impacts maintainability, not to reference documentation where comprehensiveness adds value.

### Quick Navigation

- **New to the project?** Start with Core Principles (§1) and UI Architecture (§2)
- **Writing code?** Jump to <doc:#quick-reference> (§24)
- **Refactoring?** See Implementation Priority (§22)
- **Need context?** Review related documentation below

## Topics

### Essential Reading

- <doc:ARCHITECTURE>
- <doc:DESIGN_SYSTEM>
- <doc:COLORBLIND_ACCESSIBILITY>

### Related Documentation

- <doc:MIGRATION_NOTES>
- <doc:CONTINUATION_DOCUMENT>
- <doc:DevelopmentRules-v3.2>

### Version Information

**Status:** Active  
**Last Updated:** October 26, 2025  
**Current Version:** v3.6 (Critical Fixes)  
**Previous Version:** v3.5 (Conflict Resolution)

**Changes in v3.6:**
- Fixed filename versioning (v3_5 → v3.6)
- Completed CHANGELOG entries for v3.5 and v3.6
- Fixed section reference: Quick Reference is §24, not §26
- Clarified access control: Explicit `private` required, `internal` default may be omitted
- Removed phase tracking from §22 (belongs in CONTINUATION_DOCUMENT)
- Added documentation file exception note to Rule 20
- Fixed subsection numbering consistency in §21
- Updated timestamp to current date

---

## 1. Core Principles

- **Privacy-first, local-first.** No network or sync without explicit opt-in.
- **Predictable, testable behaviors** (deterministic launch flags for tests).
- **Progressive enhancement:** Baseline iOS 17.6 / macOS 14.6; optional iOS 26 / macOS 26 features must degrade gracefully.
- **Clean architecture:** Organized, maintainable code following the structure defined in Code Organization Standards (§21).

> Note: These principles guide all technical decisions in the project.

---

## 2. UI Architecture (Protected)

> Important: These UI patterns are foundational and must not be changed without explicit user approval.

### 2.1 Master-Detail Split Screen Pattern

**Primary Navigation:** Master-Detail (NavigationSplitView) layout is the **core navigation pattern**.

**Layout:**
- **Left Pane (Master):** Item List with search, filters, and select mode
- **Right Pane (Detail):** Item Detail View showing selected item

**Platform Adaptation:**
- iPad/Mac: Split view (sidebar | detail pane)
- iPhone: Stack navigation (pushes to detail)

**Protection Level:** 🔒 **LOCKED** — Do not modify unless explicitly directed by user

### 2.2 Rationale

This pattern provides:
- Efficient workflow on larger screens (iPad, Mac)
- Quick item browsing with persistent detail view
- Consistent UX across platforms
- Professional app experience

### 2.3 Modification Policy

Any changes to the Master-Detail pattern require:
1. Explicit written approval from user
2. Documentation of reason for change
3. Update to this section with new pattern

---

## 3. Platform & Availability

- **Deployment targets:** iOS 17.6, macOS 14.6 (minimum for backward compatibility)
- **Latest versions:** iOS 26, macOS 26, visionOS 26
- **Development environment:** Xcode 26.0+

> Note: Use `@available` for any API requiring iOS > 17.6 or macOS > 14.6.

**Progressive Enhancement Guidelines:**
- Optional advanced features from iOS 18+ / macOS 15+ must be guarded with `if #available`
- Provide a no-op or alternative UI for older OS versions
- New iOS 26 / macOS 26 features should similarly degrade gracefully

---

## 4. Data Models & Schema Evolution

- SwiftData models live in `Item.swift` (multi-model file permitted until complexity dictates split)
- Every schema-affecting change requires an entry in <doc:MIGRATION_NOTES> (date, change, strategy, risk notes)
- Avoid destructive changes; prefer additive model evolution until migration strategy is formalized

> Warning: All schema changes must be documented before implementation.

---

## 5. Feature Flags

- Central enum: `AMSFeatureFlag` (add cases + doc comments) in `FeatureFlags.swift`
- Flags must have: purpose, platform requirements, temporary/permanent classification
- Temporary flags sunset within two minor versions or become permanent and undocumented

**Example:**
```swift
enum AMSFeatureFlag {
    /// Enables experimental cloud sync feature (iOS 18+)
    /// Status: Temporary - Remove by v2.5
    case cloudSync
    
    /// Enables advanced search filters
    /// Status: Permanent
    case advancedSearch
}
```

---

## 6. Launch Arguments (Test / Demo)

| Flag | Arg Pattern | Purpose | Removal Criteria |
|------|-------------|---------|------------------|
| `-in-memory-store` | flag | Ephemeral persistence for tests | Keep (core) |
| `-reset-data` | flag | Empty store for deterministic start | Keep |
| `-seed-items <n>` | value | Quickly populate UI | Remove if sample seeding replaced |
| `-fixed-date <ISO8601>` | value | Deterministic time base for tests | Keep |
| `-deterministic-add` | flag | Spaced timestamps for ordering tests | Keep (until replaced by stable sort keys) |

> Tip: Add new flags to this table with proper documentation.

---

## 7. Logging & Diagnostics

- Use `AMSLog` categories: `data`, `network`, `ui`, plus: `performance`, `migration` (add as needed)
- Redact user-entered free text with privacy annotations if expanded logging is introduced
- Introduce signposts for: launch container init, heavy fetch (> 200 items), image processing

**Example:**
```swift
AMSLog.data.info("Loaded \(items.count) items")
AMSLog.performance.debug("Image processing took \(duration)ms")
```

---

## 8. Error Handling

- Central error enum: `AMSError`
- New case: `modelInitFailed(underlying:)` for SwiftData container fallback
- No `fatalError` for recoverable paths; fallback to in-memory container + user-visible message (future UI placeholder)

> Warning: Never use `fatalError` for recoverable error conditions.

**Example:**
```swift
enum AMSError: LocalizedError {
    case modelInitFailed(underlying: Error)
    case invalidData(reason: String)
    
    var errorDescription: String? {
        switch self {
        case .modelInitFailed(let error):
            return "Database initialization failed: \(error.localizedDescription)"
        case .invalidData(let reason):
            return "Invalid data: \(reason)"
        }
    }
}
```

---

## 9. Internationalization / Localization

- Use String Catalog for new user-facing strings
- Provide plural-aware keys for day/month displays
- RTL & pseudo-localization spot check required before release tagging

**Example:**
```swift
// Use String Catalog keys
Text("item_count", count: items.count)

// For date formatting
Text(date, format: .dateTime.day().month())
```

---

## 10. Accessibility & Inclusive Design

- Follow colorblind adjustments (see <doc:COLORBLIND_ACCESSIBILITY>)
- Minimum target size 44x44 points
- Test with largest Dynamic Type size
- VoiceOver: each interactive element must have a label and, if needed, hint

> Important: Accessibility is not optional. All features must be fully accessible.

**Example:**
```swift
Button(action: deleteItem) {
    Image(systemName: "trash")
}
.accessibilityLabel("Delete item")
.accessibilityHint("Removes this item from your collection")
.frame(minWidth: 44, minHeight: 44)
```

---

## 11. Performance

- **Launch:** cold < 2s (baseline simulator reference), warm < 1s
- Use batched fetch limits + explicit sort descriptors (never rely on default ordering)
- Avoid storing large blobs directly on high-frequency list models (attachments are child models — OK)

> Note: Profile performance regularly using Instruments.

**Example:**
```swift
// Good: Explicit sort with fetch limit
let descriptor = FetchDescriptor<Item>(
    sortBy: [SortDescriptor(\.purchaseDate, order: .reverse)],
    predicate: #Predicate { $0.category == category }
)
descriptor.fetchLimit = 100
```

---

## 12. Privacy

- Maintain only one `PrivacyInfo.xcprivacy` file
- Remove duplicates (manual cleanup if tooling not available)
- Any new data collection requires documented purpose + opt-in

> Warning: All privacy manifest entries must be justified and documented.

---

## 13. Security

- No secret keys in repo
- Enable secret scanning in CI (future integration)
- TLS/pinning policy deferred until network features exist

---

## 14. Cloud / Sync (Planned)

Readiness checklist before implementing:
- Conflict resolution strategy documented
- Offline behavior + local-first reaffirmed
- User toggle & clear state disclosure

> Note: Cloud sync is planned but not yet implemented.

---

## 15. Images / Attachments

- Downscale images > 4096 px (longest edge) before storing
- Strip EXIF except orientation
- Prefer JPEG @ 0.8 or HEIC (platform default) for photos; PNG for line art

**Example:**
```swift
func processImage(_ image: UIImage) -> UIImage {
    let maxDimension: CGFloat = 4096
    guard image.size.width > maxDimension || image.size.height > maxDimension else {
        return image
    }
    
    let scale = maxDimension / max(image.size.width, image.size.height)
    let newSize = CGSize(width: image.size.width * scale, 
                         height: image.size.height * scale)
    return image.resize(to: newSize)
}
```

---

## 16. Concurrency

- UI layer types `@MainActor` when accessing mutable shared state
- Avoid unstructured detached tasks unless isolating a non-UI workload; document justification

**Example:**
```swift
@MainActor
final class ItemViewModel: ObservableObject {
    @Published var items: [Item] = []
    
    func loadItems() async {
        // Already on main actor
        items = await fetchItems()
    }
}
```

---

## 17. Testing

**Required suites:**
- Persistence: insert, delete, sort, limit
- Date logic: expiration (months+days), override, lifetime, leap year
- Relationship: attachments & extended warranty link (basic already present)
- Due soon threshold logic
- (Future) accessibility snapshots & performance metrics

> Note: Deterministic tests use `-fixed-date` or targeted override properties.

**Example:**
```swift
final class ItemTests: XCTestCase {
    func testExpirationCalculation() {
        let fixedDate = Date(timeIntervalSince1970: 1700000000)
        let item = Item(purchaseDate: fixedDate, warrantyMonths: 12)
        
        XCTAssertEqual(item.expirationDate, fixedDate.addingMonths(12))
    }
}
```

---

## 18. Documentation & Freshness

- This file updated with each rule change (bump version header + CHANGELOG)
- PRs modifying data model or launch flags must update this file and/or <doc:MIGRATION_NOTES>

> Important: Keep documentation in sync with code changes.

---

## 19. CI (Planned)

Planned stages:
- Build
- Unit tests
- UI smoke tests
- Format/lint
- Secret scan

> Note: Warnings treated as errors once initial cleanup complete.

---

## 20. Future Analytics (Deferred)

- Only anonymous aggregated metrics with explicit opt-in
- Event template: name, purpose, fields, privacy risk, retention

---

## 21. Code Organization Standards

These rules define how code files should be organized and structured for maintainability, searchability, and compile-time safety.

### 21.1 File Structure Rules

#### Rule 1: One Primary Type Per File

**What:** Each Swift file should contain exactly one primary type (class, struct, enum, actor, or protocol).

**Why:** Makes code easy to find and navigate.

**Example:**

```swift
// ✅ GOOD: Item.swift
import Foundation
import SwiftData

@Model
final class Item {
    var name: String
    var purchaseDate: Date
    // ... properties
}

// ✅ GOOD: Separate supporting types in their own files
// ItemCategory.swift
enum ItemCategory: String, Codable {
    case electronics, appliances, furniture
}
```

**Exceptions:**
- Tightly coupled helper types (e.g., `Item` and `Item.ValidationError`)
- Private nested types used only within the primary type

---

#### Rule 2: Extensions in Separate Files

**What:** Extensions should be in separate files named `TypeName+Feature.swift`.

**Why:** Keeps base types clean and organizes related functionality.

**Example:**

```swift
// Item.swift - base type
@Model
final class Item {
    var name: String
    var purchaseDate: Date
}

// Item+Warranty.swift - warranty-specific logic
extension Item {
    var isUnderWarranty: Bool {
        // warranty logic
    }
}

// Item+Export.swift - export functionality
extension Item {
    func exportToJSON() -> Data? {
        // export logic
    }
}
```

**Special Case: Cross-Cutting Utility Extensions**

When an extension provides generic utility functionality not specific to a file's primary type, extract it immediately during Phase 1 refactoring.

**Example from AllMyStuff:**

```swift
// ❌ BEFORE (in WarrantyFormView.swift):
extension Color {
    static var textFieldBackground: Color {
        #if os(macOS)
        return Color(NSColor.textBackgroundColor)
        #else
        return Color(.systemBackground)
        #endif
    }
}

struct WarrantyFormView: View {
    // ... 1000+ lines
}

// ✅ AFTER (proper extraction):
// Color+TextField.swift (in Sources/Utilities/Extensions/)
import SwiftUI

// MARK: - Text Field Background Color

extension Color {
    /// Returns an appropriate background color for text fields across platforms
    static var textFieldBackground: Color {
        #if os(macOS)
        return Color(NSColor.textBackgroundColor)
        #else
        return Color(.systemBackground)
        #endif
    }
}

// WarrantyFormView.swift (now cleaner)
struct WarrantyFormView: View {
    // ... main implementation
}
```

**Rationale:**
- Foundation type extensions (Color, String, Date) are reusable across the app
- Better discoverability in Extensions folder
- Follows Rule 1 (one primary type per file)
- No import changes needed (types already available)

---

#### Rule 3: Protocol Conformance in Extensions

**What:** Protocol conformances should be in dedicated extensions, preferably in separate files.

**Why:** Makes it clear what protocols a type conforms to and easier to find conformance implementation.

**Example:**

```swift
// Item.swift - base type
@Model
final class Item {
    var name: String
}

// Item+Identifiable.swift
extension Item: Identifiable {
    // Identifiable conformance
}

// Item+Comparable.swift
extension Item: Comparable {
    static func < (lhs: Item, rhs: Item) -> Bool {
        lhs.name < rhs.name
    }
}
```

---

#### Rule 4: Group Related Types in Folders

**What:** Related types should be grouped in folders by feature or domain.

**Why:** Makes project structure mirror app architecture.

**Example:**

```
Sources/
├── Models/
│   ├── Item/
│   │   ├── Item.swift
│   │   ├── Item+Warranty.swift
│   │   ├── Item+Export.swift
│   │   └── ItemCategory.swift
│   └── Attachment/
│       ├── Attachment.swift
│       └── Attachment+Image.swift
├── Views/
│   ├── ItemList/
│   │   ├── ItemListView.swift
│   │   └── ItemRowView.swift
│   └── ItemDetail/
│       ├── ItemDetailView.swift
│       └── WarrantyCardView.swift
```

---

#### Rule 5: Consistent Naming Conventions

**What:** Files should be named clearly and consistently.

**Patterns:**
- Base types: `TypeName.swift` (e.g., `Item.swift`)
- Extensions: `TypeName+Feature.swift` (e.g., `Item+Warranty.swift`)
- Views: `FeatureView.swift` (e.g., `ItemListView.swift`)
- View Models: `FeatureViewModel.swift` (e.g., `ItemListViewModel.swift`)

---

### 21.2 Code Organization Within Files

#### Rule 6: MARK Comments for Organization

**What:** Use `// MARK: -` comments to organize code sections within files.

**Standard sections:**
- Properties
- Initialization
- Public Methods
- Private Methods
- Protocol Conformance (if in same file)

**Example:**

```swift
final class ItemViewModel: ObservableObject {
    // MARK: - Properties
    @Published var items: [Item] = []
    
    // MARK: - Initialization
    init() {
        loadItems()
    }
    
    // MARK: - Public Methods
    func addItem(_ item: Item) {
        // ...
    }
    
    // MARK: - Private Methods
    private func loadItems() {
        // ...
    }
}
```

**For SwiftUI Views with many bindings:**

```swift
struct ComplexFormView: View {
    // MARK: - Properties
    
    // MARK: Bindings - Core Fields
    @Binding var title: String
    @Binding var date: Date
    
    // MARK: Bindings - Optional Data
    @Binding var price: Decimal?
    @Binding var notes: String
    
    // MARK: Configuration
    let isEditing: Bool
    var onSave: (() -> Void)? = nil
    
    // MARK: State - UI Control
    @State private var showingSheet = false
    @FocusState private var focusedField: Field?
    
    // MARK: State - Platform-Specific (iOS)
    #if os(iOS)
    @State private var photoItems: [PhotosPickerItem] = []
    #endif
    
    // MARK: Nested Types
    private enum Field: Hashable {
        case name, price
    }
    
    // MARK: Constants
    private let maxItems = 10
    
    // MARK: Computed Properties
    var remainingSlots: Int { maxItems - items.count }
    
    // MARK: - Body
    
    var body: some View {
        // Main view implementation
    }
    
    // MARK: - Helper Views
    
    @ViewBuilder
    private var formContent: some View {
        // Form sections
    }
    
    private func headerLabel(_ text: String) -> some View {
        // Custom label
    }
    
    // MARK: - Helper Methods
    
    private func validate() -> Bool {
        // Validation logic
    }
}
```

> Note: For views with 50+ properties, use hierarchical MARK comments (see Rule 6a) with subsections under main categories to improve navigation.

---

#### Rule 6a: MARK Strategies for Large Files (500+ lines)

**For files with 500-1000 lines:**
- Use 5-8 major MARK sections
- Each major section can have 2-4 subsections
- Leave blank line before each MARK comment

**For files with 1000+ lines:**
- Use hierarchical MARK comments
- Primary sections: `// MARK: - Section Name` (with dash)
- Subsections: `// MARK: Subsection Name` (no dash)
- Consider splitting if exceeding 1500 lines

> Warning: Files exceeding 1500 lines should be evaluated for splitting into multiple files.

**Example Hierarchy:**

```swift
// MARK: - Properties

// MARK: Bindings - Core Fields
@Binding var title: String
@Binding var date: Date
// ... (10-20 properties)

// MARK: Bindings - Additional Metadata
@Binding var notes: String
@Binding var price: Decimal?
// ... (10-20 properties)

// MARK: State - UI Control
@State private var showingSheet = false
// ... (5-10 properties)

// MARK: State - Platform-Specific
#if os(iOS)
@State private var photoItems: [PhotosPickerItem] = []
#endif

// MARK: - Body

var body: some View {
    // Implementation
}

// MARK: - Helper Views

// MARK: Form Sections
@ViewBuilder
private var basicInfoSection: some View { }

// MARK: Platform-Specific Views
#if os(iOS)
private var iosSpecificView: some View { }
#endif

// MARK: - Helper Methods

// MARK: Validation
private func validateRequired() -> Bool { }
private func validateFormat() -> Bool { }

// MARK: Data Processing
private func processImage() { }
private func formatCurrency() -> String { }
```

---

#### Rule 7: Property Organization

**What:** Organize properties in a consistent order.

**Order:**
1. `@Environment` / `@EnvironmentObject`
2. `@StateObject` / `@ObservedObject`
3. `@Binding`
4. `@State`
5. `@FocusState` / `@Namespace` / other property wrappers
6. Regular properties (let, then var)
7. Computed properties

**Example:**

```swift
struct ItemDetailView: View {
    // MARK: - Properties
    
    // Environment
    @Environment(\.modelContext) private var modelContext
    
    // Observed Objects
    @StateObject private var viewModel = ItemViewModel()
    
    // Bindings
    @Binding var selectedItem: Item?
    
    // State
    @State private var isEditing = false
    @State private var showingDeleteAlert = false
    
    // Focus
    @FocusState private var focusedField: Field?
    
    // Regular Properties
    let item: Item
    private let dateFormatter = DateFormatter()
    
    // Computed Properties
    var formattedDate: String {
        dateFormatter.string(from: item.date)
    }
}
```

---

#### Rule 8: Access Control

**What:** Use explicit access control appropriately, following Swift conventions.

**Philosophy:** Swift defaults to `internal` access. Be explicit where clarity matters.

**Levels:**
- `public` - Rarely needed (framework/library code)
- `internal` (default) - May be omitted for types/methods visible throughout the module
- `fileprivate` - Use sparingly for file-scoped sharing
- `private` - Always explicit for implementation details

**When to be explicit:**
- ✅ Always use `private` for implementation details
- ✅ Always use `public` for APIs meant to be external
- ✅ Use `fileprivate` when needed for file-scoped access
- ⚠️ Use `internal` explicitly when it improves code searchability or clarifies intent
- ❌ Avoid cluttering code with unnecessary `internal` keywords on every declaration

**Example:**

```swift
// ✅ GOOD: Explicit where it matters
struct ItemListView: View {
    @Environment(\.modelContext) private var modelContext
    private let dateFormatter = DateFormatter()
    
    var body: some View {
        List {
            // ...
        }
    }
    
    private func deleteItem(_ item: Item) {
        // ...
    }
}

// ✅ ALSO ACCEPTABLE: Explicit internal for searchability
internal struct ItemListView: View {
    @Environment(\.modelContext) private var modelContext
    private let dateFormatter = DateFormatter()
    
    internal var body: some View {
        List {
            // ...
        }
    }
    
    private func deleteItem(_ item: Item) {
        // ...
    }
}
```

> Tip: Use `private` liberally for implementation details. Use `internal` explicitly when it improves code clarity or aids in project-wide searches.

---

### 21.3 SwiftUI-Specific Rules

#### Rule 9: Extract Complex Views

**What:** Extract view logic into separate computed properties or custom views when body gets complex.

**Threshold:** If `body` exceeds ~50 lines, extract sections.

**Example:**

```swift
// ❌ BAD: 200-line body
struct ItemDetailView: View {
    var body: some View {
        ScrollView {
            VStack {
                // 200 lines of view code
            }
        }
    }
}

// ✅ GOOD: Extracted sections
struct ItemDetailView: View {
    var body: some View {
        ScrollView {
            VStack {
                headerSection
                infoSection
                warrantySection
                actionsSection
            }
        }
    }
    
    // MARK: - Helper Views
    
    @ViewBuilder
    private var headerSection: some View {
        // Header implementation
    }
    
    @ViewBuilder
    private var infoSection: some View {
        // Info implementation
    }
    
    @ViewBuilder
    private var warrantySection: some View {
        // Warranty implementation
    }
    
    @ViewBuilder
    private var actionsSection: some View {
        // Actions implementation
    }
}
```

---

#### Rule 10: Consistent View Modifiers

**What:** Apply view modifiers in a consistent order.

**Recommended order:**
1. Layout (frame, padding)
2. Appearance (background, foreground, font)
3. Behavior (disabled, onTapGesture)
4. Accessibility
5. Navigation

**Example:**

```swift
Button("Delete") {
    deleteItem()
}
.frame(minWidth: 44, minHeight: 44)
.padding()
.background(Color.red)
.foregroundColor(.white)
.cornerRadius(8)
.disabled(isDeleting)
.accessibilityLabel("Delete item")
.accessibilityHint("Removes this item permanently")
```

---

#### Rule 11: Preview Organization

**What:** Organize previews with clear examples.

**Example:**

```swift
#Preview("Default State") {
    ItemDetailView(item: .sample)
}

#Preview("Empty State") {
    ItemDetailView(item: .empty)
}

#Preview("Loading State") {
    ItemDetailView(item: .sample)
        .environment(\.isLoading, true)
}

#Preview("Dark Mode") {
    ItemDetailView(item: .sample)
        .preferredColorScheme(.dark)
}
```

---

#### Rule 12: View Modifiers as Extensions

**What:** Create reusable view modifiers for common styling patterns.

**Example:**

```swift
// ViewModifiers.swift
extension View {
    func cardStyle() -> some View {
        self
            .padding()
            .background(Color(.systemBackground))
            .cornerRadius(12)
            .shadow(radius: 2)
    }
}

// Usage
Text("Hello")
    .cardStyle()
```

---

### 21.4 Testing Organization

#### Rule 13: Test File Organization

**What:** Mirror source structure in test files.

**Structure:**
```
Tests/
├── Models/
│   └── ItemTests.swift
├── ViewModels/
│   └── ItemViewModelTests.swift
└── Utilities/
    └── DateHelperTests.swift
```

---

#### Rule 14: Test Naming

**What:** Use descriptive test names that explain what is being tested.

**Pattern:** `test[What]_[Condition]_[ExpectedResult]`

**Example:**

```swift
final class ItemTests: XCTestCase {
    func testWarrantyStatus_WhenWithinPeriod_ReturnsActive() {
        // Arrange
        let item = Item(purchaseDate: Date(), warrantyMonths: 12)
        
        // Act
        let status = item.warrantyStatus
        
        // Assert
        XCTAssertEqual(status, .active)
    }
    
    func testWarrantyStatus_WhenExpired_ReturnsExpired() {
        // Arrange
        let pastDate = Calendar.current.date(byAdding: .year, value: -2, to: Date())!
        let item = Item(purchaseDate: pastDate, warrantyMonths: 12)
        
        // Act
        let status = item.warrantyStatus
        
        // Assert
        XCTAssertEqual(status, .expired)
    }
}
```

---

#### Rule 15: MARK in Tests

**What:** Use MARK comments to organize test suites by functionality.

**Example:**

```swift
final class ItemTests: XCTestCase {
    // MARK: - Setup
    
    override func setUp() {
        super.setUp()
        // Test setup
    }
    
    // MARK: - Initialization Tests
    
    func testInit_WithValidData_CreatesItem() { }
    func testInit_WithNilOptionals_UsesDefaults() { }
    
    // MARK: - Warranty Tests
    
    func testWarrantyExpiration_WithStandardPeriod_CalculatesCorrectly() { }
    func testWarrantyExpiration_WithLeapYear_HandlesCorrectly() { }
    
    // MARK: - Relationship Tests
    
    func testAttachments_WhenAdded_MaintainsRelationship() { }
    func testExtendedWarranty_WhenLinked_UpdatesExpiration() { }
}
```

---

### 21.5 Documentation

#### Rule 16: Public API Documentation

**What:** All public types and methods must have documentation comments.

**Format:** Use `///` for documentation comments.

**Example:**

```swift
/// Represents a physical item with warranty information
///
/// Use this model to track items you own, including their purchase
/// information, warranty details, and associated attachments.
@Model
final class Item {
    /// The display name of the item
    var name: String
    
    /// The date the item was purchased
    var purchaseDate: Date
    
    /// Duration of the manufacturer's warranty in months
    var warrantyMonths: Int
    
    /// Calculates whether the item is currently under warranty
    ///
    /// - Returns: `true` if the current date is within the warranty period
    var isUnderWarranty: Bool {
        let expirationDate = calculateExpirationDate()
        return Date() <= expirationDate
    }
}
```

---

#### Rule 17: Inline Comments

**What:** Use inline comments sparingly and only for complex logic.

**Guidelines:**
- Explain *why*, not *what*
- Avoid obvious comments
- Update comments when code changes

**Example:**

```swift
// ✅ GOOD: Explains reasoning
// Use a Set for O(1) lookup when checking duplicates
let uniqueItems = Set(items)

// ❌ BAD: States the obvious
// Create a set from items
let uniqueItems = Set(items)

// ✅ GOOD: Explains non-obvious behavior
// SwiftData requires explicit save after batch deletes
// to ensure proper relationship cleanup
try? modelContext.save()
```

---

#### Rule 18: TODO and FIXME

**What:** Use standardized markers for code that needs attention.

**Markers:**
- `// TODO:` - Features to implement
- `// FIXME:` - Bugs to fix
- `// MARK: - Refactor` - Code that needs restructuring

**Example:**

```swift
// TODO: Add support for recurring warranty renewals
// FIXME: Handle nil date edge case (crashes on iOS 17.0)
// TODO: Extract validation logic to separate service
```

---

### 21.6 Error Handling

#### Rule 19: Consistent Error Handling

**What:** Use a centralized error enum and handle errors consistently.

**Example:**

```swift
enum AMSError: LocalizedError {
    case invalidInput(field: String)
    case databaseError(underlying: Error)
    case notFound(itemName: String)
    
    var errorDescription: String? {
        switch self {
        case .invalidInput(let field):
            return "Invalid input for field: \(field)"
        case .databaseError(let error):
            return "Database error: \(error.localizedDescription)"
        case .notFound(let name):
            return "Item '\(name)' not found"
        }
    }
    
    var recoverySuggestion: String? {
        switch self {
        case .invalidInput:
            return "Please check your input and try again"
        case .databaseError:
            return "Try restarting the app"
        case .notFound:
            return "The item may have been deleted"
        }
    }
}
```

---

#### Rule 20: File Length Limits

**What:** Keep implementation files under 1500 lines for maintainability.

**When to split:**
- File exceeds 1500 lines
- Section is >300 lines and logically independent
- Functionality is reusable elsewhere
- Testing becomes difficult

**Exception:** Reference documentation files (like this document) may exceed this limit when comprehensiveness adds value. The 1500-line guideline applies to code files where complexity impacts maintainability, not to documentation where length aids reference.

**Example:**

```swift
// ❌ BAD: 2000-line view controller
final class ItemViewController {
    // Too much code in one file
}

// ✅ GOOD: Split into focused files
// ItemViewController.swift (main logic - 800 lines)
// ItemViewController+DataSource.swift (table view - 300 lines)
// ItemViewController+Delegates.swift (delegate methods - 200 lines)
```

---

### 21.7 Optionals

#### Rule 21: Prefer Optional Chaining

**What:** Use optional chaining and nil coalescing over forced unwrapping.

**Example:**

```swift
// ❌ BAD: Force unwrapping
let name = item.name!
let email = user.email!.lowercased()

// ✅ GOOD: Optional chaining with default
let name = item.name ?? "Unknown"
let email = user.email?.lowercased() ?? ""

// ✅ GOOD: Guard for early exit
guard let name = item.name else {
    return
}
processName(name)
```

---

#### Rule 22: Use Optional Binding Pattern

**What:** Use if-let or guard-let for safe unwrapping.

**Example:**

```swift
// ✅ GOOD: If-let
if let item = selectedItem {
    displayItem(item)
}

// ✅ GOOD: Guard-let for early exit
guard let item = selectedItem else {
    showEmptyState()
    return
}
displayItem(item)

// ✅ GOOD: Multiple bindings
if let item = selectedItem,
   let warranty = item.warranty,
   warranty.isActive {
    showWarrantyInfo(warranty)
}
```

---

### 21.8 Collections

#### Rule 23: Use Appropriate Collection Types

**What:** Choose the right collection type for your use case.

**Guidelines:**
- `Array` - Ordered collection with duplicates
- `Set` - Unordered, unique elements, fast lookup
- `Dictionary` - Key-value pairs, fast lookup by key

**Example:**

```swift
// ✅ GOOD: Array for ordered list
let items: [Item] = fetchItems()

// ✅ GOOD: Set for unique categories
let categories: Set<String> = Set(items.map { $0.category })

// ✅ GOOD: Dictionary for lookup by ID
let itemsByID: [UUID: Item] = Dictionary(
    uniqueKeysWithValues: items.map { ($0.id, $0) }
)
```

---

#### Rule 24: Use Higher-Order Functions

**What:** Prefer map, filter, reduce over manual loops when appropriate.

**Example:**

```swift
// ❌ BAD: Manual loop
var activeItems: [Item] = []
for item in items {
    if item.isActive {
        activeItems.append(item)
    }
}

// ✅ GOOD: Filter
let activeItems = items.filter { $0.isActive }

// ✅ GOOD: Map
let itemNames = items.map { $0.name }

// ✅ GOOD: CompactMap (removes nils)
let emails = users.compactMap { $0.email }

// ✅ GOOD: Reduce
let totalPrice = items.reduce(0) { $0 + $1.price }
```

---

### 21.9 Performance

#### Rule 25: Lazy Loading

**What:** Use lazy properties for expensive computations that may not be needed.

**Example:**

```swift
final class ItemViewModel {
    let item: Item
    
    // Computed each time it's accessed
    var expensiveValue: String {
        performExpensiveCalculation()
    }
    
    // Computed once, then cached
    lazy var cachedExpensiveValue: String = {
        performExpensiveCalculation()
    }()
}
```

---

#### Rule 26: Batch Operations

**What:** When working with collections, prefer batch operations over individual operations in loops.

**Example:**

```swift
// ❌ BAD: Individual operations
for item in items {
    modelContext.delete(item)
    try? modelContext.save()
}

// ✅ GOOD: Batch operation
items.forEach { modelContext.delete($0) }
try? modelContext.save()

// ✅ EVEN BETTER: Batch delete if available
try? modelContext.delete(model: Item.self, where: predicate)
```

---

## 22. Implementation Priority

When applying code organization rules to existing code, follow this recommended order. Track actual refactoring progress in <doc:CONTINUATION_DOCUMENT>, not in this permanent guideline document.

### Priority 1: Critical Structure
- **Rule 8:** Add explicit access control where needed
- **Rule 1:** Ensure one primary type per file
- **Rule 6:** Add MARK comments for navigation

### Priority 2: Improved Organization
- **Rules 2 & 3:** Extract extensions to separate files
- **Rule 4:** Reorganize into feature-based folders
- **Rule 7:** Organize properties consistently within types

### Priority 3: Enhanced Quality
- **Rule 16:** Add documentation to public APIs
- **Rules 13 & 14:** Improve test organization and naming
- **Rule 9:** Extract complex SwiftUI views

### Priority 4: Ongoing Maintenance
- Apply all rules to new code immediately
- Refactor existing code opportunistically (when touching a file for other reasons)

> Tip: Don't refactor everything at once. Focus on files you're already modifying.

> Note: For tracking which files have been refactored and current phase status, see <doc:CONTINUATION_DOCUMENT>.

---

## 23. Naming Consistency

### Current Naming Standards

All code prefixes use **AMS** (AllMyStuff) for consistency:
- `AMSLog` - Logging utilities
- `AMSError` - Error handling
- `AMSFeatureFlag` - Feature flags

### Implementation Notes
- AMS prefixes are consistently applied throughout the codebase
- Any legacy prefixes found should be updated to AMS during refactoring
- Document any legacy naming in code comments if update would cause disruption

---

## 24. Quick Reference {#quick-reference}

### Access Control Quick Guide

- `public` - Exported to other modules (rare in this app)
- `internal` (default) - Available within app (may omit for brevity)
- `fileprivate` - Same file only (rare - prefer private)
- `private` - Same type only (always explicit)

> Tip: Use `private` for implementation details. Omit or use `internal` for app-wide visibility based on code clarity needs.

### MARK Comment Patterns

**Standard View Structure:**
```swift
// MARK: - Properties
// MARK: - Body
// MARK: - Helper Views
// MARK: - Helper Methods
```

**Standard Model Structure:**
```swift
// MARK: - Properties
// MARK: - Initialization
// MARK: - Public Methods
// MARK: - Private Methods
```

**Standard ViewModel Structure:**
```swift
// MARK: - Properties
// MARK: - Initialization
// MARK: - Actions
// MARK: - Private Helpers
```

**Large SwiftUI View (50+ properties):**
```swift
// MARK: - Properties
  // MARK: Bindings - Core Fields
  // MARK: Bindings - Metadata
  // MARK: State - UI Control
  // MARK: State - Platform-Specific
  // MARK: Configuration
  // MARK: Constants
  // MARK: Computed Properties
// MARK: - Body
// MARK: - Helper Views
// MARK: - Helper Methods
```

### File Naming Quick Guide

- Extensions: `TypeName+Feature.swift`
- Tests: `TypeNameTests.swift`
- Views: `FeatureNameView.swift`
- ViewModels: `FeatureNameViewModel.swift`
- Models: `EntityName.swift`
- Utilities: `DescriptiveName.swift` (in Utilities folder)

### When to Extract

**Extract extension if:**
- It's a utility/helper not specific to the primary type
- It could be reused in other files
- It's protocol conformance (separate file)
- File has multiple concerns

**Extract to new file if:**
- File exceeds 1500 lines
- Section is >300 lines and logically independent
- Functionality is reusable elsewhere
- Testing becomes difficult

### Implementation Phase Cheat Sheet

**Phase 1 (Critical Structure):**
- Add `private` to implementation details
- Extract utility extensions (Color, String, etc.)
- Add 5-8 MARK sections (15+ for large files)

**Phase 2 (Improved Organization):**
- Extract protocol conformances to extensions
- Split large files (>1500 lines)
- Reorganize into feature-based folders

**Phase 3 (Enhanced Quality):**
- Add documentation comments (`///`)
- Improve test coverage and organization
- Extract complex SwiftUI views into components

**Phase 4 (Ongoing):**
- Apply all rules to new code immediately
- Refactor opportunistically when modifying files

---

## 25. Using This Document

### For New Features

1. Review relevant sections (especially §1-3, §21)
2. Follow code organization rules (§21) from the start
3. Update CHANGELOG if adding new patterns

### For Refactoring

1. Use Implementation Priority (§22) to guide efforts
2. Don't refactor everything at once
3. Focus on files you're already modifying

### For Code Review

1. Check compliance with core principles (§1)
2. Verify code organization rules (§21)
3. Ensure documentation is updated (§18)

---

## 26. CHANGELOG

- **v3.6 (2025-10-26):**
  - **Critical fixes:** Renamed file from v3_5 to v3.6 for version consistency
  - **Section references:** Fixed Quick Reference from §26 to correct §24
  - **Access control clarity:** Clarified that `private` is always explicit, `internal` may be omitted or used based on code clarity needs
  - **Phase tracking:** Removed from §22; belongs in CONTINUATION_DOCUMENT for temporary project state
  - **Documentation exception:** Added explicit note to Rule 20 about reference documentation file size
  - **Subsection numbering:** Fixed consistency in §21 rule numbering
  - **Timestamp:** Updated to October 26, 2025
  - **CHANGELOG completion:** Added missing v3.5 entry and this v3.6 entry

- **v3.5 (2025-10-21):**
  - **Documentation size exception:** Added clarification that 1500-line rule applies to code, not reference docs
  - **Access control update:** Aligned guidance with Swift conventions (explicit where it matters)
  - **Verbosity reduction:** Clarified when `internal` keyword should be explicit vs omitted

- **v3.4 (2025-10-20):**
  - **Practical examples:** Enhanced Rule 6 with SwiftUI View MARK pattern showing 50+ binding organization
  - **Extraction guidance:** Added utility extension pattern to Rule 2 (Color+TextField example)
  - **Large file strategy:** Added Rule 6a for 1000+ line files with hierarchical MARK guidance
  - **Quick reference:** Added §24 with access control, MARK patterns, naming, and phase guides
  - **Maintenance:** Added "Last Updated" timestamp to header
  - **Fixes:** Corrected section references
  - **File management:** Created DevelopmentRules-v3.4.md (proper version alignment)

- **v3.3 (2025-10-20):**
  - **Quality improvements:** Removed redundant rename notes
  - **Consistency fix:** Renumbered subsections under §21
  - **Terminology cleanup:** Changed "Product Item List" to "Item List"
  - **Result:** Improved clarity and consistency throughout document

- **v3.2 (2025-10-20):**
  - Updated for Xcode 26, iOS 26, macOS 26
  - Simplified naming consistency section (§23)
  - Removed legacy project name references

- **v3.1 (2025-10-19):**
  - Added §2 (UI Architecture - Protected)
  - Established Master-Detail split screen pattern as protected architectural foundation
  - Added Select Mode functionality to Item List (bulk deletion with Select All)
  - Renumbered subsequent sections
  - Documented requirement for explicit user approval before modifying core UI patterns

- **v3.0 (2025-10-13):**
  - Merged with code organization standards (26 rules)
  - Added Code Organization Standards section
  - Added Implementation Priority section
  - Added Naming Consistency Update section
  - Established AMS naming conventions

- **v2.2:** Converted to Markdown; added feature flags, model init fallback, testing expansion, migration notes protocol, image policy, duplicate privacy manifest note.

- **2025-10-05:** Removed stray duplicate file `PrivacyInfo 2.xcprivacy` (canonical manifest retained as `PrivacyInfo.xcprivacy`).

---

**This document is the single source of truth for AllMyStuff development standards.**
