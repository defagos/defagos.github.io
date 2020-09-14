# Porting UIKit Collections to SwiftUI

When faced with the task of implementing a collection with `UICollectionView`-like behavior, two options can be considered:

- Writing a collection view with behaviors similar to `UICollectionView` entirely in SwiftUI.
- Wrapping `UICollectionView` into a SwifUI view using `UIViewRepresentable`.

Since `UICollectionView` is robust and supports all the features we could dream of, the second option is the obvious choice. Still we want our collection to be used and feel like a SwiftUI view, which means:

- It must nicely integrate with SwiftUI declarative formalism.
- Cells and supplementary views must be built in SwiftUI, not in UIKit, and certainly not with xibs or constraint-based layouts.
- Cells and supplementary views must be lazily instantiated and reused. Put another way, performance must be similar to what is usually expected from `UICollectionView`.
- The collection view must support arbitrary layouts and nicely update when SwiftUI detects a change it needs to apply.
- Focus must be correctly managed on tvOS, in particular it should be possible to implement the standard user experience expected on Apple TV according to the HIG.

With these constraints in mind, let's start building our SwiftUI collection.

## Wrapping UICollectionView into SwiftUI

What makes SwiftUI so appealing in the first place is the ease with which you can embed UIKit views in SwiftUI, or conversely. This makes it easy to adopt SwiftUI where you can in your app, while still using good old UIKit where SwiftUI is lagging behind in functionality or performance.

SwiftUI provides the `UIViewRepresentable` protocol to wrap a UIKit view into SwiftUI. We could also wrap a view controller into a `UIViewControllerRepresentable`, but I prefer the former approach since there is here no real advantage in wrapping `UICollectionViewController` instead of `UICollectionView` directly.

If you are not familiar with SwiftUI, just recall that `View`s in SwiftUI can be seen as short-lived _snapshots of a layout_, unlike `UIView`s in UIKit which are _content actually drawn on screen_. When describing your layout in SwiftUI, you therefore provide an up-to-date snapshot of your layout which then gets applied by the SwiftUI layout engine, leading to content actually being drawn or updated on screen. View snapshots are then discarded until new ones must be produced for further updates.

When interfacing UIKit with SwiftUI this important distiction translates into the two protocol methods required by `UIViewRepresentable`:

- `makeUIView(context:)` is called when the `UIView` described by a `View` needs to be created for the first time.
- `updateUIView(_:context)` is called when an already created `UIView` described by a `View` is updated.

We therefore start our implementation by conforming to `UIViewRepresentable` and implementing its two required methods:

```
struct CollectionView: UIViewRepresentable {
    func makeUIView(context: Context) -> some UIView {
        // Create the collection view for the first time
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        // Update the existing collection view
    }
}
```

In our case  the actual `UICollectionView` instance must be created in `makeUIView(context:)`, but this requires a collection view layout to be provided at initialization time.

## Providing a Collection View Layout

With iOS 13 and tvOS 13 collection view layouts can be provided in a general declarative way through compositional layouts. The obtained layout code is both expressive and supports powerful features, among them independently scrollable sections. If you remember this is exactly how we wanted our tvOS layout to behave in the first place. 

Because SwiftUI is also available starting with iOS 13 and tvOS 13, compositional layouts are the obvious choice for our implementation:

```
struct CollectionView: UIViewRepresentable {
    private func layout() -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            // Return section layout
        }
    }

    func makeUIView(context: Context) -> some UIView {
        return UICollectionView(frame: .zero, collectionViewLayout: layout())
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        // Update the existing collection view
    }
}
```

Note that we provide `.zero` as frame since SwiftUI will later apply a suitable frame when the view is actually on screen.

We will later see that compositional layouts are defined on a section basis, each section containing groups of items, with supplementary or decoration views, and possibly independent orthogonal scrolling behavior. How and where this layout definition is provided is yet still unclear, and for the moment hidden in the `layout()` method.

Before we proceed with layout definition further, let's first discuss how data is loaded into the collection. This will provide us with very interesting insights about our collection and the API it should provide.

## Loading Data Into the Collection

Since iOS 13 and tvOS 13, and in addition to compositional layouts, UIKit provides a `UICollectionViewDiffableDataSource` to incrementally update data associated with a collection. More importantly this data source ensures that cells and underlying data stay consistent, avoiding crashes ususally associated with mismatches between data and layout. It also knows how to animate changes for a smoother experience.

When a data change needs to be made, a snapshot must be created and applied to the data source, which then takes care of the rest. This in fact fits quite nicely with how SwiftUI reacts to data changes in general: When a change is detected, we provide a new layout description to represent the data in its new state, and SwiftUI takes care of the rest. 

For our collection view, a parent content view which represents the data from some `Model` conforming to `ObservableObject` would roughly be implemented as follows:

```
struct ContentView: View {
    @ObservedObject var model: Model
    
    private func data(from model: Model) -> CollectionData {
        // Build data as expected by the CollectionView
    }

    var body: some View {
        CollectionView(data: data(from: model))
    }
}
```

When a model update is published by the observed object, a private method creates a new data snapshot in some format understood by the `CollectionView`, yet to be determined. The SwiftUI layout engine then either creates or update the underlying `UICollectionView` using this data snapshot.

What kind of data format should we use? Internally we want to rely on `UICollectionViewDiffableDataSource`, a generic type parametrized with two types, section and item, both required to be `Hashable`. With the similar way compositional layouts are defined (sections containing groups of items), it is therefore natural to model the data displayed by a `CollectionView` as rows of items, each row being generically described by section and item types:

```
struct CollectionRow<Section: Hashable, Item: Hashable>: Hashable {
    let section: Section
    let items: [Item]
}
```

Note that since `Section` and `Item` both are `Hashable`, making `CollectionViewRow` hashable only costs an explicit protocol conformance. Ensuring rows are hashable will prove useful to quickly check whether a row has changed, but more on that later.

This new type introduced, the collection view itself must be a generic type parametrized by `Section` and `Item`:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    let rows: [CollectionRow<Section, Item>]
    
    private func layout() -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            // Return section layout
        }
    }

    func makeUIView(context: Context) -> some UIView {
        return UICollectionView(frame: .zero, collectionViewLayout: layout())
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        // Update the existing collection view
    }
}
```

The `rows` are provided when a new `CollectionView` layout snapshot needs to be applied. Types for the generic parameters will be automatically be inferred by the Swift compiler in general. Still we must provide rows to the underlying `UICollectionView` data source, as we will see in the next section.

## Data Source and View Coordinator

A `UICollectionView` requires a data source to provide the data it displays. Instantiating a diffable data source and associating it with a `collectionView` instance of `UICollectionView` is straightforward:

```
let dataSource = UICollectionViewDiffableDataSource<Section, Item>(collectionView: collectionView) { collectionView, indexPath, item in
    // Return UICollectionViewCell for the item
}
```

This data source must be retained, as the collection view only contains a weak reference to it, automatically associated with the `UICollectionViewDiffableDataSource`. But where should we store a strong reference to the data source to keep it alive, since `View`s in SwiftUI are merely short-lived snapshots which gets destroyed once they have been applied?

Fortunately SwiftUI provides an answer in the form of coordinators. The `UIViewRepresentable` protocol namely lets you optionally implement a `makeCoordinator()` method, from which you can return a custom instance. SwiftUI calls this method before creating the UIKit view from the first time and associates the coordinator with it, keeping the coordinator alive for as long as the `UIView` is in use.

We don't know much about our coordinator type yet, except that it must store our data source. After initial creation, this coordinator is provided to the `makeUIView(context:)` method through the context parameter, but as a constant. We must mutate set the contained data source reference when the `UICollectionView` as been instantiated (the diffable data source requires the collection view at initialization time), so our `Coordinator` is required to be a `class`:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    class Coordinator {
        fileprivate typealias DataSource = UICollectionViewDiffableDataSource<Section, Item>
        
        fileprivate var dataSource: DataSource? = nil
    }
    
    // ...
    
    func makeCoordinator() -> Coordinator {
        return Coordinator()
    }
    
    func makeUIView(context: Context) -> some UIView {
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout())
        context.coordinator.dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            // Return UICollectionViewCell for the item
        }
        return collectionView
    }
}
```

Now that we have a data source properly setup and retained for the lifetime of the collection view, we can discuss how data is inserted into it to be displayed by the collection in the end.

## Data Source Snapshots

Updating a diffable data source is simple. It suffices to build a new data snapshot and apply it:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    // ...
    
    private func snapshot() -> NSDiffableDataSourceSnapshot<Section, Item> {
        var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
        for row in rows {
            snapshot.appendSections([row.section])
            snapshot.appendItems(row.items, toSection: row.section)
        }
        return snapshot
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        guard let dataSource = context.coordinator.dataSource else { return }
        dataSource.apply(snapshot(), animatingDifferences: true)
    }
}
```

There are two performance issues associated with the above code:

- The first applied snapshot will be slow if animated.
- If you run this code on tvOS for a large number of rows items and try to move around the collection, performance is poor. If you try the same code on iOS performance is fine.

To understand the second performance issue, we must remember that `updateUIView(_:context:)` is called:

- When a source of truth changed in a parent `View` (e.g. an `@ObservedObject` or `@State` changed).
- When the `@Environment` changed (e.g. the screen was rotated or the font size was changed).

On tvOS the focus is part of the environment, so whenever the focus is moved around, an environment change is triggered, leading to a full view update with the naive implementation above. This explains why moving around the collection on tvOS can be slower than on iOS and why you should in general avoid performing time-consuming work in `updateUIView(_:context:)`.

You might remember that in the first article in this series I conjectured this might be the reason why SwiftUI nested stacks and scroll views are much slower on tvOS than on iOS (and the reason why this series of articles exists in the first place). Though SwiftUI stacks and scroll views are not implemented in UIKit as far as I know, it is possible that too much layout work is currently performed as a result of the focus being moved on tvOS.

The next section focuses on solving these two performance issues.

## Solving Performance Issues

The first performance problem we are facing can be solved by applying the initial snapshot without animation. We factor the corresponding code into a dedicated `reloadData(context:animated)` method:

```
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    // ...
    
    private func reloadData(context: Context, animated: Bool = false) {
        guard let dataSource = context.coordinator.dataSource else { return }
        dataSource.apply(snapshot(), animatingDifferences: animated)
    }
    
    func makeUIView(context: Context) -> some UIView {
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout())
        
        context.coordinator.dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            // Return UICollectionViewCell for the item
        }
        
        reloadData(context: context)
        return collectionView
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        reloadData(context: context, animated: true)
    }
}
```

The second performance problem we are facing on tvOS is due to snapshots being applied too many times, in particular when the focus is moved. There is currently no way to apply updates outside `updateUIView(_:context:)`, though, and no API to distinguish between updates due to data or environment changes. Can we still avoid calculating and applying snapshots when not necessary? How can we know that data has not changed?

It is actually quite simple. As you may remember we made `CollectionViewRow` conform to `Hashable`. Calculating hashes is quite cheap, and so is calculating the hash of an array of hashable instances. Each time we applied a snapshot we can store a hash of its `rows`, persist this value into our coordinator so that it stays available between updates, and only apply a new snapshot if the hash changes:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    class Coordinator {
        // ...
        
        fileprivate var rowsHash: Int? = nil
    }
    
    // ...
    
    struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    // ...
    
    private func reloadData(context: Context, animated: Bool = false) {
        let coordinator = context.coordinator
        guard let dataSource = coordinator.dataSource else { return }
        
        let rowsHash = rows.hashValue
        if coordinator.rowsHash != rowsHash {
            dataSource.apply(snapshot(), animatingDifferences: animated)
            coordinator.rowsHash = rowsHash
        }
    }
}
```

With this simple optimization performance problems on tvOS are covered, even with large data sets. Note that though the `CollectionView` inhibit reloads due to environment changes, subviews like cells still can implement proper response to environment changes if they need to.

We now almost have a first working `CollectionView` implementation, but we still need to be able to configure its layout and cells when creating it.

## Collection Layout API

At the beginning of this article we instantiated a `UICollectionViewCompositionalLayout` but did not provide any meaningful implementation:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    // ...

    private func layout() -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            // Return section layout
        }
    }
}
```

Now is finally the time to fill this gap.

How should our API look like? We want our collection to behave as a native SwiftUI component, it would therefore be tempting to write a layout using function builders to have a SwiftUI-inspired syntax for layout construction too. IMHO this is a bit too much, as the current compositional layout API is already declarative and simple enough. Wrapping everything in SwiftUI-like syntax would not provide many readability benefits, only more code.

Our compositional layout internally requires a single section layout provider trailing closure, so this is what our external API will let the user customize:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    // ...
    
    let rows: [CollectionRow<Section, Item>]
    let sectionLayoutProvider: (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection

    private func layout() -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            return sectionLayoutProvider(sectionIndex, layoutEnvironment)
        }
    }
}
```

The layout for each section is defined when building a `CollectionView`, for example a shelf-like layout like the one we implemented in SwiftUI in the first part of this article series:

```
CollectionView(rows: rows()) { sectionIndex, layoutEnvironment in
    let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1), heightDimension: .fractionalHeight(1))
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
            
    let groupSize = NSCollectionLayoutSize(widthDimension: .absolute(320), heightDimension: .absolute(180))
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])
            
    let section = NSCollectionLayoutSection(group: group)
    section.orthogonalScrollingBehavior = .continuous
    return section
}
```

Note that `rows()` construction is here not important for the layout itself and has been omitted.

The layout example above is quite simple. In general each section layout might depend on the data model, though, for example if the layout is received from a web service and belongs to the model itself. In such cases the `sectionLayoutProvider` block will capture the initial context and keep it for subsequent updates, which is not what we want if the layout returned by the web service later changes. To solve this issue our friend the coordinator again comes to the rescue:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    class Coordinator {
        // ...
        
        fileprivate var sectionLayoutProvider: ((Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection)?
    }

    let rows: [CollectionRow<Section, Item>]
    let sectionLayoutProvider: (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection

    private func layout(context: Context) -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            return context.coordinator.sectionLayoutProvider!(sectionIndex, layoutEnvironment)
        }
    }
    
    private func reloadData(context: Context, animated: Bool = false) {
        let coordinator = context.coordinator
        coordinator.sectionLayoutProvider = self.sectionLayoutProvider
        
        guard let dataSource = coordinator.dataSource else { return }
        
        let rowsHash = rows.hashValue
        if coordinator.rowsHash != rowsHash {
            dataSource.apply(snapshot(), animatingDifferences: animated)
            coordinator.rowsHash = rowsHash
        }
    }
        
    // ...
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        reloadData(context: context, animated: true)
    }
}
```

This way the section layout provider is updated between view updates and the layout is always the most recent one if derived from the data model. Note that the optional `sectionLayoutProvider` stored by the `Coordinator` can here be safely force unwrapped, as it will always be available when layout is triggered by a view update. We also updated the `layout(context:)` method with a `Context` parameter.


## Cell Layout API

Our `CollectionView` first implementation is almost finished. The cell provider closure of our `UICollectionViewDiffableDataSource` is namely still missing. Recall that we don't want to layout cells with UIKit code but with SwiftUI declarative syntax. How should we now proceed?

SwiftUI syntax is built on top of function builders, more precisely view builders. To be able to associate SwiftUI cells with their `UICollectionViewCell` counterparts (which we still need internally for our `UICollectionView` implementation), we simply ensure our view builder signature mirrors the one of `UICollectionViewDiffableDataSource` cell provider:

```
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    // ...
    
    let rows: [CollectionRow<Section, Item>]
    let sectionLayoutProvider: (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection
    let cell: (IndexPath, Item) -> Cell
    
    init(rows: [CollectionRow<Section, Item>],
         sectionLayoutProvider: @escaping (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection,
         @ViewBuilder cell: @escaping (IndexPath, Item) -> Cell) {
        self.rows = rows
        self.sectionLayoutProvider = sectionLayoutProvider
        self.cell = cell
    }
}
```

To benefit from `@ViewBuilder` syntax we must implement a dedicated initializer, as the `@ViewBuilder` attribute can only appear as a parameter-attribute. Note that Swift infers the type of the cell body based on the view builder block, forcing us to add `Cell` as additional parameter of the `CollectionView` generic type, conforming to the `View` protocol.

## SwiftUI Cell Hosting

With the API we just designed clients of our `CollectionView` provide SwiftUI views as cells. Embedding SwiftUI views into UIKit is made by using `UIHostingController`. To display a SwiftUI cell witin a `UICollectionViewCell`s, all we need is therefore a cell host:

```
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    // ...
    
    private class HostCell: UICollectionViewCell {
        private var hostController: UIHostingController<Cell>?
        
        override func prepareForReuse() {
            if let hostView = hostController?.view {
                hostView.removeFromSuperview()
            }
            hostController = nil
        }
        
        var hostedCell: Cell? {
            willSet {
                guard let view = newValue else { return }
                hostController = UIHostingController(rootView: view)
                if let hostView = hostController?.view {
                    hostView.frame = contentView.bounds
                    hostView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
                    contentView.addSubview(hostView)
                }
            }
        }
    }
}
```

The implementation of this host cell is quite straightforward:

- We prepare a `UIHostingController` when setting a SwiftUI cell. The hosting controller view is installed to fill the entire cell content view. Note that creating a `UIHostingController` for a SwiftUI is cheap enough for this approach only to require a single kind of cell.
- When a cell is reused, the hosting controller view is removed and the controller itself released.

Now that we have a cell, it suffices to register it and return dequeued instances from our data source:

```
struct CollectionView<Section: Hashable, Item: Hashable>: UIViewRepresentable {
    // ...
    
    func makeUIView(context: Context) -> some UIView {
        let cellIdentifier = "hostCell"
    
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout(context: context))
        collectionView.register(HostCell.self, forCellWithReuseIdentifier: cellIdentifier)
        
        context.coordinator.dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            let hostCell = collectionView.dequeueReusableCell(withReuseIdentifier: cellIdentifier, for: indexPath) as? HostCell
            hostCell?.hostedCell = cell(indexPath, item)
            return hostCell
        }
        
        reloadData(context: context)
        return collectionView
    }
}
```

### Remark

iOS and tvOS 14 provide a new `CellRegistration` API but, according to my tests in recent beta versions, they are simply not reusing cells correctly (`prepareForReuse` is never called). The code above therefore uses the old cell registration and dequeuing API.

## Complete implementation

We have now implemented all bits and pieces of our `CollectionView`, whose complete implementation is listed below:

```
import SwiftUI

struct CollectionRow<Section: Hashable, Item: Hashable>: Hashable {
    let section: Section
    let items: [Item]
}

struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    private class HostCell: UICollectionViewCell {
            private var hostController: UIHostingController<Cell>?
            
            override func prepareForReuse() {
                if let hostView = hostController?.view {
                    hostView.removeFromSuperview()
                }
                hostController = nil
            }
            
            var hostedCell: Cell? {
                willSet {
                    guard let view = newValue else { return }
                    hostController = UIHostingController(rootView: view)
                    if let hostView = hostController?.view {
                        hostView.frame = contentView.bounds
                        hostView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
                        contentView.addSubview(hostView)
                    }
                }
            }
        }
    
    class Coordinator {
        fileprivate typealias DataSource = UICollectionViewDiffableDataSource<Section, Item>
        
        fileprivate var dataSource: DataSource? = nil
        fileprivate var sectionLayoutProvider: ((Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection)?
        fileprivate var rowsHash: Int? = nil
    }
    
    let rows: [CollectionRow<Section, Item>]
    let sectionLayoutProvider: (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection
    let cell: (IndexPath, Item) -> Cell
    
    init(rows: [CollectionRow<Section, Item>],
         sectionLayoutProvider: @escaping (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection,
         @ViewBuilder cell: @escaping (IndexPath, Item) -> Cell) {
        self.rows = rows
        self.sectionLayoutProvider = sectionLayoutProvider
        self.cell = cell
    }
    
    private func layout(context: Context) -> UICollectionViewLayout {
        return UICollectionViewCompositionalLayout { sectionIndex, layoutEnvironment in
            return context.coordinator.sectionLayoutProvider!(sectionIndex, layoutEnvironment)
        }
    }
    
    private func snapshot() -> NSDiffableDataSourceSnapshot<Section, Item> {
        var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
        for row in rows {
            snapshot.appendSections([row.section])
            snapshot.appendItems(row.items, toSection: row.section)
        }
        return snapshot
    }
    
    private func reloadData(context: Context, animated: Bool = false) {
        let coordinator = context.coordinator
        coordinator.sectionLayoutProvider = self.sectionLayoutProvider
        
        guard let dataSource = coordinator.dataSource else { return }
        
        let rowsHash = rows.hashValue
        if coordinator.rowsHash != rowsHash {
            dataSource.apply(snapshot(), animatingDifferences: animated)
            coordinator.rowsHash = rowsHash
        }
    }
    
    func makeCoordinator() -> Coordinator {
        return Coordinator()
    }
    
    func makeUIView(context: Context) -> some UIView {
        let cellIdentifier = "hostCell"
        
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout(context: context))
        collectionView.register(HostCell.self, forCellWithReuseIdentifier: cellIdentifier)
        
        context.coordinator.dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            let hostCell = collectionView.dequeueReusableCell(withReuseIdentifier: cellIdentifier, for: indexPath) as? HostCell
            hostCell?.hostedCell = cell(indexPath, item)
            return hostCell
        }
        
        reloadData(context: context)
        return collectionView
    }
    
    func updateUIView(_ uiView: UIViewType, context: Context) {
        reloadData(context: context, animated: true)
    }
}
```

Using this collection view we can implement a shelf-like tvOS layout like we did in the first part of this article:

```
import SwiftUI

struct Shelf: View {
    typealias Row = CollectionRow<Int, String>
    
    var rows: [Row] = {
        var rows = [Row]()
        for i in 0..<20 {
            rows.append(Row(section: i, items: (0..<20).map { "\(i), \($0)" }))
        }
        return rows
    }()
    
    var body: some View {
        CollectionView(rows: rows) { sectionIndex, layoutEnvironment in
            let itemSize = NSCollectionLayoutSize(widthDimension: .fractionalWidth(1), heightDimension: .fractionalHeight(1))
            let item = NSCollectionLayoutItem(layoutSize: itemSize)
            
            let groupSize = NSCollectionLayoutSize(widthDimension: .absolute(320), heightDimension: .absolute(180))
            let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])
            
            let section = NSCollectionLayoutSection(group: group)
            section.contentInsets = NSDirectionalEdgeInsets(top: 20, leading: 0, bottom: 20, trailing: 0)
            section.interGroupSpacing = 40
            section.orthogonalScrollingBehavior = .continuous
            return section
        } cell: { indexPath, item in
            GeometryReader { geometry in
                Button(action: {}) {
                    Text(item)
                        .frame(width: geometry.size.width, height: geometry.size.height)
                        .background(Color.blue)
                }
                .buttonStyle(CardButtonStyle())
            }
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .ignoresSafeArea(.all)
    }
}
```

Once you run the application you will quickly find a few issues with the result:

- When scrolling horizontally appearaing cells are not properly sized.
- Buttons do not exhibit the `CardButtonStyle` appearance like they should.

The final article in this series will discuss how these issues can be solved, and cover supplementary views as well as decoration views for improved tvOS navigation.

