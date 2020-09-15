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

In the shelf example discussed in part 2 of this series, using the new tvOS 14 `CardButtonStyle`, the expected focused appearance is not applied when running the application. This is actually expected behavior, as the cell itself is focusable and the button style [is not expected to work in such cases](https://developer.apple.com/videos/play/wwdc2020/10042).

Since our `CollectionView` is a SwiftUI component, we don't need to use any delegate for cell selection and should handle such cases with simple buttons. This means our host cells should not be focusable:

```
private class HostCell: UICollectionViewCell {
    // ...
        
    override var canBecomeFocused: Bool {
         return false
    }
}
```

This simple change alone fixes focused card button appearance:

![Fixed focus](/images/collection_fixed_focus.jpg)

Also note that SwiftUI correctly restores the recently focused item when returning from modal presentation, which is pretty nice.

### Remark

During my investigations I investigated focusable cell behavior more extensively, so here are a few more details if you are interested. 

As said above, using focusable cells disables focused button appearance. But in fact it even prevents their use (their action is simply not triggered), making SwiftUI `Button`s wrapped in focusable cells basically useless.

Moreover, any desired focused appearance must then be implemented manually. Scaling is easy, tilting is another matter, though, and the native tvOS look & feel is not easy to reproduce perfectly, with the problem you might never get the exact same result as Apple official implementation. 

Finally, the pressed appearance (obtained for free with `CardButtonStyle`) requires catching cell interactions and transferring them up the SwiftUI view hierarchy, for example within the `@Environment` using a custom `EnvironmentKey`. Again this requires the appearance (scaling most notably) to be adjusted manually.

For all these reasons I recommend to never make cells focusable if you intend to wrap SwiftUI views within them.

## Supporting Supplementary Views

Supporting supplementary views is very similar to supporting cells. We simply introduce a dedicated view builder and a host view. As for cells, type inference requires the addition of a new `SupplementaryView `type parameter to the generic `CollectionView` type, as well as a corresponding host view and view builder closure:

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

Supplementary views are registered for specific kinds (e.g. header, footer or custom kind) and a single reuse identifier, as for cells. We store known supplementary view kinds in our coordinator as they are registered:
   
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
        
        reloadData(context: context)
        return collectionView
    }
    
    // ...
}
```

## Further Possible Improvements

This implementation can without a doubt be further improved. I hope it gave you some intuition about how UIKit views can be embedded into SwiftUI in non-trivial cases, so that you can embed other views in a similar way should you find an issue that cannot be directly solved in SwiftUI.

The following exercices are left for the reader:

- Improve the public API with more expressive block signatures.
- Implement supplementary view support in an optional way.
- Use decoration views to add focus guides for navigation between rows of different length on tvOS.

This concludes this series of articles. Hope you enjoyed the ride!
