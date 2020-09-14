# Part 3: A SwiftUI Collection Polishing

In part 2 of this article series we implemented a first working collection view in SwiftUI, based on `UICollectionView`. This implementation is promising but still has a few issues and shortcomings that we will address in this final article.

## Fixing Cell Frames

When running our shelf example it appears that cells on screen have correct frames, while cells coming from screen edges don't:

![SwiftUI CollectionView](/images/first_swiftui_collection.jpg)

Having a look at `UICollectionViewCell` and `UIHostingController` view frames, the problem is found to be related to how SwiftUI assigns a frame to views, unexpectedly applying safe area insets when such a controller is used for simple view embedding. 

After searching for a bit, a [Stack Overflow thread](https://stackoverflow.com/questions/61552497/uitableviewheaderfooterview-with-swiftui-content-getting-automatic-safe-area-ins) proposes a workaround until `UIHostingController` itself provides an official API to disable this behavior. This workaround applies swizzling to all hosting view instances indiscriminately to ignore safe area insets, a serious issue for hosting controllers for which this behavior is actually desired (e.g. the `UIHostingController` containing the root SwiftUI view of the application, if any).

Instead of swizzling, we can provide the missing opt-in as an extension on `UIHostingController`, applied by dynamically subclassing the hosting view:

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

With this fix cell frames are now correct:

![Fixed frame](/images/collection_fixed_cell_size.jpg)

## Supporting Focus on tvOS

In our shelf example from previous article, using the new tvOS 14 `CardButtonStyle`, the expected focused appearance is not applied when running the project. Tthis effect does not work because the collection cell itself is focused and the button style [does not work in such cases](https://developer.apple.com/videos/play/wwdc2020/10042).

Since our `CollectionView` is a SwiftUI component, we don't need to use any delegate for cell selection and should handle such cases with simple buttons. This means our host cells should not be focusable:

```
private class HostCell: UICollectionViewCell {
    // ...
        
    override var canBecomeFocused: Bool {
         return false
    }
}
```

This simple change alone suffices to fix focused button appearance:

![Fixed focus](/images/collection_fixed_focus.jpg)

Also note that SwiftUI correctly restores the focused item when returning from modal presentation, which is pretty nice.

### Remark

During my investigations I attempted to make cells focusable. As said above this approach disables focused button appearance and even prevents their use (their action is simply not triggered). 

If we want a focused appearance it must also be implemented manually. Scaling is easy, tilting is another matter, though, and the native look & feel is not easy to reproduce perfectly. Moreover, the pressed appearance (obtained for free with `CardButtonStyle`) requires catching cell interactions and transferring them up the SwiftUI view hierarchy, e.g. with a custom environment key.

For all these reasons disabling focus for cells embedding SwiftUI views is the easiest approach to get correct behavior consistent with the usual tvOS user experience.

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

Supplementary views are registered for specific kinds (e.g. header, footer or custom kind). We store known kinds of views in our coordinator as they are registered:
   
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

This implementation can without a doubt be further improved. I hope it gave you some intuition about how UIKit views can be embedded into SwiftUI in non-trivial cases, so that you can embed other views in a similar way, should you find an issue that cannot be directly solved in SwiftUI.

The following exercices are left for the reader:

- Improve the public API with more expressive block signatures and optional supplementary view support.
- Use decoration views to add focus guides for navigation between rows of different length on tvOS.


This concludes this series of articles. Hope you enjoyed the ride!
