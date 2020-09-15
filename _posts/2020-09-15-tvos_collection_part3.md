# Part 3: A SwiftUI Collection Polishing

In part 2 of this article series we implemented a first working collection view in SwiftUI, powered by `UICollectionView`. This implementation is promising but still has a few issues and shortcomings that we will address in this final article.

## Fixing Cell Frames

When running our shelf example cells initially on screen have correct frames, while cells merging from screen edges do not:

![SwiftUI CollectionView](/images/first_swiftui_collection.jpg)

When inspected in the view debugger, `UICollectionViewCell` and `UIHostingController` view frames are fine, so the problem must be related to how SwiftUI assigns a frame to views contained in a `UIHostingController`. In fact, closer inspection of the applied frames reveals that the reduction in size is due to safe area insets being somehow applied.

A [Stack Overflow thread](https://stackoverflow.com/questions/61552497/uitableviewheaderfooterview-with-swiftui-content-getting-automatic-safe-area-ins) proposes a workaround for this issue until `UIHostingController` itself provides an official API to disable this behavior. This workaround applies swizzling to all hosting view instances indiscriminately, disabling safe area inset support entirely for all hosted SwiftUI views. 

This approach is too greedy and affects hosting controllers for which this behavior is actually legitimate and desired, for example `UIHostingController` hosting the SwiftUI root of a `UIApplication`. A more surgical approach than method swizzling is to use [dynamic subclassing](https://funwithobjc.tumblr.com/post/1482787069/dynamic-subclassing), the runtime hackery applied by key-value observing most notably. To sketch the idea briefly, consider we have an object of some class whose behavior we want to tweak:

- First create a subclass of this class.
- Add methods to the subcass (possibly calling a parent implementation if this makes sens).
- Change the class of the object to the subclass.

In our case, we can provide the missing opt-in as an extension on `UIHostingController`, applied by dynamically changing the hosting view class with a subclass ignoring safe area insets:

```
extension UIHostingController {
    convenience public init(rootView: Content, ignoreSafeArea: Bool) {
        self.init(rootView: rootView)
        
        if ignoreSafeArea {
            disableSafeArea()
        }
    }
    
    func disableSafeArea() {
        guard let viewClass = object_getClass(view) else { return }
        
        let viewSubclassName = String(cString: class_getName(viewClass)).appending("_IgnoreSafeArea")
        if let viewSubclass = NSClassFromString(viewSubclassName) {
            object_setClass(view, viewSubclass)
        }
        else {
            guard let viewClassNameUtf8 = (viewSubclassName as NSString).utf8String else { return }
            guard let viewSubclass = objc_allocateClassPair(viewClass, viewClassNameUtf8, 0) else { return }
            
            if let method = class_getInstanceMethod(UIView.self, #selector(getter: UIView.safeAreaInsets)) {
                let safeAreaInsets: @convention(block) (AnyObject) -> UIEdgeInsets = { _ in
                    return .zero
                }
                class_addMethod(viewSubclass, #selector(getter: UIView.safeAreaInsets), imp_implementationWithBlock(safeAreaInsets), method_getTypeEncoding(method))
            }
            
            objc_registerClassPair(viewSubclass)
            object_setClass(view, viewSubclass)
        }
    }
}
```

This way we only apply a change of behavior to those `UIHostingController` instances for which safe area insets must not be taken into account, for example in our `HostCell` implementation:

```
hostController = UIHostingController(rootView: view, ignoreSafeArea: true)
```

With this simple trick cell frames are now correct:

![Fixed frame](/images/collection_fixed_cell_size.jpg)

## Supporting Focus on tvOS

In the shelf example discussed in part 2 of this series, using the new tvOS 14 `CardButtonStyle`, the usual focused appearance is not applied when running the application. This is actually expected behavior, as the cell itself is focusable and the button style [is not meant to receive focus in such cases](https://developer.apple.com/videos/play/wwdc2020/10042).

In fact, SwiftUI buttons wrapped in focusable cells cannot be triggered at all. As a good SwiftUI citizen, our `CollectionView` cannot exclude prevent buttons from being used in cells, therefore host cells must not be focusable. This means we will not respond to collection cell standard selection delegate, but let buttons handle actions.

One way to disable focus for a cell is by implementing `canBecomeFocused` on the cell class itself and return `false`. This works well in general, except when data source changes are applied with animations, in which case the focus is likely to spin out of control. We therefore need a better strategy.

First we need to understand what the standard expected behavior is. To find the answer I implemented a separate basic `UICollectionView` with a simple data source and focusable `UICollectionViewCell`s. I could observe that the focused item is followed when data source changes are animated and that the focus never spins out of control in such cases. There is still a minor issue with the focused appearance being lost after the animation ends (likely a `UICollectionView` bug), but it suffices to swipe the remote again to fix the issue. This user experience sets our goal for our SwiftUI `CollectionView` focus behavior.

All our problems are related to the fact we disabled focus entirely for cells. We cannot enable focus globally for reasons mentioned above, still we can enable it when an animated reload is made. This can be achieved thanks to `UICollectionViewDelegate`, which provides a dedicated delegate method to decide at any time whether a cell must be focusable or not. We therefore make our `Coordinator` conform to this protocol and introduce an internal flag to enable or disable focus at any time for cells:

```
public struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    public class Coordinator: NSObject, UICollectionViewDelegate {
        // ...
        
        fileprivate var focusable: Bool = false
        
        public func collectionView(_ collectionView: UICollectionView, canFocusItemAt indexPath: IndexPath) -> Bool {
            return focusable
        }
    }
}
```

We can now enable `UICollectionView` cell focus right when the collection needs it, letting it correctly track the currently focused item. To keep things as local as possible, we enable focus, force a focus update, and then immediately reset the flag to its nominal value:

```
public struct CollectionView<Section: Hashable, Item: Hashable, Cell: View>: UIViewRepresentable {
    private func reloadData(in collectionView: UICollectionView, context: Context, animated: Bool = false) {
        let coordinator = context.coordinator
        coordinator.sectionLayoutProvider = self.sectionLayoutProvider
        
        guard let dataSource = coordinator.dataSource else { return }
        
        let rowsHash = rows.hashValue
        if coordinator.rowsHash != rowsHash {
            dataSource.apply(snapshot(), animatingDifferences: animated) {
                coordinator.focusable = true
                collectionView.setNeedsFocusUpdate()
                collectionView.updateFocusIfNeeded()
                coordinator.focusable = false
            }
            coordinator.rowsHash = rowsHash
        }
    }
}
```

Note that this requires the `UICollectionView` to be provided as additional parameter to our reload method.

With these changes the focus now appears as expected and the focused item is followed when the data source is updated:

![Fixed focus](/images/collection_fixed_focus.jpg)

### Remark

During my investigations I could have a deeper look at focusable cell behavior. Here are a few more details if you are interested. 

It is possible to keep cells focusable but, as said above, `Button`s are then basically useless. Moreover, focused appearance must be implemented manually by tweaking view properties. Scaling is easy but tilting is another matter, and the native tvOS look & feel is not easy to reproduce accurately. 

Finally, the pressed appearance (obtained for free with `CardButtonStyle`) requires catching cell interactions and transferring them up the SwiftUI view hierarchy, for example through the `@Environment` using a custom `EnvironmentKey`. Again this requires the appearance (scaling) to be adjusted manually.

For all these reasons I recommend to avoid making cells focusable if you intend to wrap SwiftUI views within them.

## Supporting Supplementary Views

Supporting supplementary views is very similar to supporting cells. We simply introduce a dedicated view builder and a host view. As for cells, type inference requires the addition of a new `SupplementaryView `type parameter to the generic `CollectionView` type:

```
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View, SupplementaryView: View>: UIViewRepresentable {
    private class HostSupplementaryView: UICollectionReusableView {
        private var hostController: UIHostingController<SupplementaryView>?
        
        override func prepareForReuse() {
            if let hostView = hostController?.view {
                hostView.removeFromSuperview()
            }
            hostController = nil
        }
        
        var hostedSupplementaryView: SupplementaryView? {
            willSet {
                guard let view = newValue else { return }
                hostController = UIHostingController(rootView: view, ignoreSafeArea: true)
                if let hostView = hostController?.view {
                    hostView.frame = self.bounds
                    hostView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
                    addSubview(hostView)
                }
            }
        }
    }
    
    // ...
    
    let supplementaryView: (String, IndexPath) -> SupplementaryView
    
    init(rows: [CollectionRow<Section, Item>],
         sectionLayoutProvider: @escaping (Int, NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection,
         @ViewBuilder cell: @escaping (IndexPath, Item) -> Cell,
         @ViewBuilder supplementaryView: @escaping (String, IndexPath) -> SupplementaryView) {
        self.rows = rows
        self.sectionLayoutProvider = sectionLayoutProvider
        self.cell = cell
        self.supplementaryView = supplementaryView
    }
}
```

Supplementary views are registered for a single reuse identifier (for the same reason a single identifier is required for cells) but specific kinds (e.g. header, footer or custom kind). We store known kinds in our coordinator as they are registered so that each required registration is made at most once:
   
```
struct CollectionView<Section: Hashable, Item: Hashable, Cell: View, SupplementaryView: View>: UIViewRepresentable {
    // ...
    
    class Coordinator: NSObject, UICollectionViewDelegate {
        // ...
        fileprivate var registeredSupplementaryViewKinds: [String] = []
    }

    func makeUIView(context: Context) -> UICollectionView {
        let cellIdentifier = "hostCell"
        let supplementaryViewIdentifier = "hostSupplementaryView"
        
        let collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout(context: context))
        collectionView.register(HostCell.self, forCellWithReuseIdentifier: cellIdentifier)
        
        let dataSource = Coordinator.DataSource(collectionView: collectionView) { collectionView, indexPath, item in
            let hostCell = collectionView.dequeueReusableCell(withReuseIdentifier: cellIdentifier, for: indexPath) as? HostCell
            hostCell?.hostedCell = cell(indexPath, item)
            return hostCell
        }
        context.coordinator.dataSource = dataSource
        
        dataSource.supplementaryViewProvider = { collectionView, kind, indexPath in
            let coordinator = context.coordinator
            if !coordinator.registeredSupplementaryViewKinds.contains(kind) {
                collectionView.register(HostSupplementaryView.self, forSupplementaryViewOfKind: kind, withReuseIdentifier: supplementaryViewIdentifier)
                coordinator.registeredSupplementaryViewKinds.append(kind)
            }
            
            guard let view = collectionView.dequeueReusableSupplementaryView(ofKind: kind, withReuseIdentifier: supplementaryViewIdentifier, for: indexPath) as? HostSupplementaryView else { return nil }
            view.hostedSupplementaryView = supplementaryView(kind, indexPath)
            return view
        }
        
        reloadData(in:collectionView, context: context)
        return collectionView
    }
    
    // ...
}
```

## Additional Considerations

Before we wrap things up I just wanted to mention a few important behaviors related to our implementation:

- Since cells are reused any `@State` they store is temporary, If you need more persistent state it should probably be stored in your model instead.
- Diffable data sources need each item to have a unique identifier, no matter the row it appears in, otherwise error will be reported at runtime. This is because in general items can be moved between rows. If several of your rows need to display the same item without having it moving between sections, just associate each item with its section.

Further improvements could be made and are left as exercise for the reader:

- Make supplementary view support optional.
- Make tabs optionally scroll with the content on tvOS (Hint: `tabBarObservedScrollView`).
- Use decoration views to add focus guides for navigation between rows of different length on tvOS.

## Wrapping Things Up

This article is the last one in this series. I hope you enjoyed the ride and learned a few things along the way!

The code for the collection view itself and a few examples is available on GitHub. I even added a SPM manifest so that you can quickly test the collection in a project of your own. This does not mean the code is intended to be used as a library yet, but feel free to use it any way you want.
