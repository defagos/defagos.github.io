# A SwiftUI Collection Romance

In its first iteration SwiftUI did not offer any native collection view like UIKit does, though it was possible to create fairly complex collection-like layouts using a combination of nested stacks and scroll views. In its second iteration, SwiftUI intends to further close the gap with UIKit by introducing lazy stack variants as well as new lazy grid components.

Building grid layouts in SwiftUI is incredibly simple. If you haven't, just try and you'll be amazed how simple and beautiful the formalism is. And with the additions made this year, there hasn't been any better time to fully embrace SwiftUI, has there?

That's what I initially thought and, when considering our options for porting our existing apps from iOS to tvOS, we naturally decided to use SwiftUI, especially since we had a working iOS SwiftUI prototype and we knew that tvOS is a simpler ecosystem in general.

But as usual things didn't turn out quite exactly as I imagined in the first place.

## Love at First Sight

Think about the TV+, App Store or Netflix apps. It is fairly common to present content in horizontally scrollable shelves, and coincidentally this is how we present content in our apps.

How would you write such a layout in SwiftUI? Well, last year you would simply nest a few stacks and scroll views, and _voilà_:

```
struct Cell: View {
    let row: Int
    let column: Int
    
    var body: some View {
        Text("\(row), \(column)")
            .frame(width: 320, height: 180)
            .background(Color.blue)
    }
}

struct Row: View {
    let index: Int
    
    var body: some View {
        ScrollView(.horizontal) {
            HStack {
                ForEach(0..<10) { i in
                    Cell(row: index, column: i)
                }
            }
        }
    }
}

struct Grid: View {
    var body: some View {
        ScrollView {
            VStack {
                ForEach(0..<20) { i in
                    Row(index: i)
                }
            }
        }
    }
}
```

![Stack grid](/images/stack_grid.jpg)

Achieving a similar result with UIKit would have required a `UICollectionView` and a compositional layout with horizontally scrollable sections, requiring much more code to write. And this assumes your application targets iOS or tvOS 13 and above: On earlier versions you would probably have used a main vertical collection or table, and as many nested horizontal collections as rows. Not to mention that in this case horizontal content offsets have to be saved and restored as the main vertical collection or table is scrolled and the cells reused.

The real downside of this approach is that, unlike `UICollectionView`, all view bodies are loaded initially, as is appearant when you profile the code with Instruments:

![Instruments stack](/images/instruments_stack.jpg)

## Romance

The greatness does not stop there. SwiftUI introduces lazy variants of stacks in iOS and tvOS 14. Seems like magic when you watch [WWDC 2020 10031](https://developer.apple.com/wwdc20/10031) where they are introduced, but you still have to be somewhat careful about which stack you promote to laziness (note this is one of the rare cases where laziness can be considered a motive for promotion).

In our case only the outermost stack should be made lazy so that each row height can be properly calculated by the layout engine:

```
struct Cell: View {
    let row: Int
    let column: Int
    
    var body: some View {
        Button(action: {}) {
            Text("\(row), \(column)")
                .frame(width: 320, height: 180)
                .background(Color.blue)
        }
    }
}

struct Row: View {
    let index: Int
    
    var body: some View {
        ScrollView(.horizontal) {
            HStack {
                ForEach(0..<10) { i in
                    Cell(row: index, column: i)
                }
            }
        }
    }
}

struct Grid: View {
    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(0..<20) { i in
                    Row(index: i)
                }
            }
        }
    }
}
```

![Lazy stack grid](/images/lazy_stack_grid.jpg)

When profiled with Instruments, only relevant view bodie are considered:

![Instruments stack](/images/instruments_lazy_stack.jpg)

That though iOS and tvOS 14 also introduce lazy grids, with similar behavior, those cannot be used for our layout, as they currently do not support independently scrollable rows.

## Drama

I had never been happier since I discovered my new friends the SwiftUI stacks and scroll views. But now was time to scroll the content, and this is where our romance ended.

What is not immediately apparant if you don't run the code samples above on tvOS is that you cannot actually scroll the content. There is namely no [focusable](https://developer.apple.com/design/human-interface-guidelines/tvos/app-architecture/focus-and-selection) item, therefore no way to navigate the collection with the Apple TV remote.

Fortunately it is very easy to make cells focusable by turning them into buttons. We can even have standard look and feel with the new tvOS 14 card button style, which makes focused buttons pop out, with a large shadow underneath and the ability to tilt the view by gently touching the remote.

Since focus makes the button larger and adds a large shadow to it, I tweaked the margins to let the content shine when focused, but the rest is quite the same, except for the added button wrapper:

```
struct Cell: View {
    let row: Int
    let column: Int
    
    var body: some View {
        Button(action: {}) {
            Text("\(row), \(column)")
                .frame(width: 320, height: 180)
                .background(Color.blue)
        }
        .buttonStyle(CardButtonStyle())
    }
}

struct Row: View {
    let index: Int
    
    var body: some View {
        ScrollView(.horizontal) {
            HStack {
                ForEach(0..<10) { i in
                    Cell(row: index, column: i)
                }
            }
            .padding([.leading, .trailing], 40)
            .padding(.top, 20)
            .padding(.bottom, 80)
        }
    }
}

struct Grid: View {
    var body: some View {
        ScrollView {
            VStack {
                ForEach(0..<20) { i in
                    Row(index: i)
                }
            }
        }
    }
}
```

I could now navigate the collection and enjoy the native tvOS behavior I was expecting, but the experience felt a bit sluggish overall.

![Lazy stack grid](/images/focusable_grid.jpg)

As it is not uncommon for our app to present 20 shelves of 20 items, I got a bit greedy and attempted to load 20 rows of 50 columns. I know how important collections are in the iOS and tvOS ecosystems, and I took for granted that SwiftUI would internally perform all necessary optimizations to have great performance when nesting stacks. It turns out this is not the case at the moment, and performance clearly degrades as the number of items increases.

Furthermore, when using a `LazyVStack` for the outermost stack, navigation on tvOS is especially cumbersome when reaching the screen edges, as the focus can only be moved one row at a time.

Note that if you run the same code on iOS (without the card button style available on tvOS only), the experience is better overall:

- Scrolling is smoother, which does not mean that it is anywhere the performance of a `UICollectionView` displaying the same number of items. Without having the time to dig into what actually makes things worse on tvOS, I conjecture that focus changes on tvOS trigger `@Environment` changes which lead to additional layout work, something we will discuss again in a further article. If this happens to be the case we can probably expect performance improvements in the future.
- There is no issue when reaching lazy stack boundaries, as there is no focus.

On iOS, you can also experience slight view pop-in at screen boundaries when scrolling fast.

## Breakup

To achieve the required layout with a great user experience, nested stack and scroll views are not appropriate and fail to deliver a satisfying user experience scaling well with increased number of items. This is where I had to choose the attitude to adopt next:

- Blissful optimism: Do nothing, continue to work with nested stack and scroll views, and hope that later iOS and tvOS betas fix the issue.
- Complete pessimism: Consider SwiftUI is not mature enough and just use UIKit.
- Moderation: Keep SwiftUI, but find a way to solve collection performance issues.

Having tasted how great SwiftUI can be, I decided to combine `UICollectionView` performance and view reuse with SwiftUI declarative layouts, by wrapping a standard `UICollectionView` into SwiftUI.

Read next: 

