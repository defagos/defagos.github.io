---
layout: post
title: Understanding SwiftUI Layout Behaviors
---

The SwiftUI layout system is intended to be more predictable, easier to understand and use than UIKit layout system. But in practice the situation is often a bit more complicated.

For newcomers with no preconception of how layout historically worked on Apple platforms, official documentation about the SwiftUI layout system is namely incomplete or [obscure](https://developer.apple.com/documentation/SwiftUI/View/frame(minWidth:idealWidth:maxWidth:minHeight:idealHeight:maxHeight:alignment:)). The number of views and modifiers, as well as their various behaviors, can be quite overwhelming. Even for seasoned UIKit developers it can be difficult to figure out how SwiftUI layout system works, as its core principles are quite different from UIKit well-known concepts of Auto Layout constraints, springs and struts.

This article explores the essential rules and behaviors of the SwiftUI layout system and explain how you should reason about it. It also introduces a formalism that helps characterize views and their sizing behaviors in general. It finally provides a complete reference of the sizing behavior for most SwiftUI built-in views.

* TOC
{:toc}

## View Categories

SwiftUI views all belong to either one of the following categories:

- **Simple views** which do not take another view (or view builder) as parameter, like `Text` or `Image`.
- **Composed views** which take another view or a view builder as parameter, like `VStack` or `Toggle`.

Layouts themselves are simply created by assembling simple views and composed views in arbitrarily complex hierarchies.

## The SwiftUI Layout Process

When a parent must lay out one of its child views it proceeds in three steps, very well explained in articles from [Alex Grebenyuk](https://kean.blog/post/swiftui-layout-system) and [Paul Hudson](https://www.hackingwithswift.com/books/ios-swiftui/how-layout-works-in-swiftui):

1. The parent offers some size to the child view.
2. The child view decides the size it requires, possibly taking into account the parent size offer (as a hint which the child is free to ignore entirely). It then returns the size it requires to its parent.
3. The parent lays out the child where it wants, strictly respecting the size that the child requested.

### Examples
{:.no_toc}

Size offer and child view placements can have various forms, for example:

- A `Painting` draws some ostentatious painting frame around its edges, and offers the rest to a single child view centered in it.
- A `Border` offers its entire size to a single child view and adds a 1-pixel border on top of it.
- A `Plane` is made of two coordinate axes and equally offers a fourth of its size to four child views, each one drawn in a quadrant. Child views are positioned in their respective quadrant with different alignments (bottom left for the 1st, bottom right for the 2nd, top right for the 3rd and top left for the 4th), so that one of their corners is located at the origin.

## Sizing Behaviors

During step 2 of the layout process children must decide the size they need before communicating it to their parent. We call *sizing behaviors*[^1] the various possibilities with which a view can decide the size it needs. Note that views might have different sizing behaviors in the horizontal and vertical directions. Knowing which sizing behavior is adopted by a view and how it affects the layout process is crucial in understanding how SwiftUI assigns sizes and positions to views.

Usually a view exhibits one of the following opposite sizing behaviors:

- **Expanding** (exp): The view strives to match the size offered by its parent.
- **Hugging** (hug): The view decides of the best size to fit its content without consulting the size offered by its parent.

If a view is expanding in a single direction it must match the size offered by its parent in this direction so that it can scale when the parent does. But if a view expands in horizontal and vertical directions at the same time it must only fulfill the size offered by its parent in at least one direction.[^2] Child views are namely ultimately responsible of deciding alone which size they want to take; If they are expanding in all directions they must still match the size offered by their parent in at least one direction (so that they can scale when the parent does), but they remain free not to match the size in the other direction it they do not want to stretch their content.[^3]

A third behavior must be introduced for composed views, whose behavior depend on their children behavior:

- **Chameleon** (cha): The view adjusts its sizing behavior based on the behaviors of its children, adopting hugging behavior if and only if all its children have hugging behavior, otherwise expanding behavior.

These three sizing behaviors are *intrinsic*, which means they apply to views considered in isolation. In concrete situations, though, views are part of a layout, for which chameleon behavior always translates into expanding or hugging behavior. Chameleon behavior must therefore be seen as a behavioral placeholder for expanding or hugging behaviors.

In the following we might sometimes use h-exp, v-exp, h-hug, v-hug, h-cha and v-cha as shorthands for all possible intrinsic behaviors in horizontal (h) and vertical (v) directions respectively.

[^1]: The term _sizing behavior_ is encountered in [Apple frame documentation](https://developer.apple.com/documentation/SwiftUI/View/frame(minWidth:idealWidth:maxWidth:minHeight:idealHeight:maxHeight:alignment:)) without proper definition.
[^2]: This is why the definition of expanding behavior mentions the view _strives to match_ and not _matches_ the size of its parent.
[^3]: This is for example how the aspect ratio modifier works for resizable content.

## Decorator Views and Modifiers

Composed views are containing a single child are special. They namely behave like decorators, possibly altering or preserving the decorated view behavior depending on their purpose:

- If a decorator has hugging behavior in some direction it can contraint its child to some size in this direction.
- If a decorator has expanding behavior in some direction it can provide its child with more space in this direction.
- If a decorator has chameleon behavior in some direction it transparently adopts the behavior of its child in the same direction.

SwiftUI makes extensive use of decorators for defining modifiers. Each modifier has an associated private composed view wrapper, returned as an opaque type from the view modifier. Common examples include `View/frame(width:height:)` or `View/aspectRatio(_:contentMode:)`. Modifiers are therefore synctactic sugar for view wrappers, nothing more.

## Determining the Intrinsic Sizing Behavior of a View

You can probe the sizing behavior of a view to determine its intrinsic sizing behavior, even if you don't have access to its implementation. The procedure to follow depends on the category the view belongs to.

### Probing Intrinsic Sizing Behavior for Simple Views
{:.no_toc}

To determine the sizing behavior of a simple view use a sufficiently large canvas, attach a border to the simple view, and observe where the border is displayed:

{% highlight swift linenos %}
struct SimpleView_Previews: PreviewProvider {
    static var previews: some View {
        SimpleView(...)
            .border(Color.blue)
            .previewLayout(.fixed(width: 1000, height: 1000))
    }
}
{% endhighlight %}

If the border is close to the simple view for some direction it has hugging behavior in this direction, otherwise expanding behavior.

TODO with Text for example + screenshots

### Probing Intrinsic Sizing Behavior for Composed Views
{:.no_toc}

To determine the behavior of a composed view use a sufficiently large canvas, attach a border to the composed view, and observe where the border is displayed when the composed view wraps an expanding child view (like `Color`), respectively a hugging view (like `Text`):[^4]

{% highlight swift linenos %}
struct ComposedView_Previews: PreviewProvider {
    static var previews: some View {
        Group {
            ComposedView(...) {
                Color.red
            }
            ComposedView(...) {
                Text("Test")
            }
        }
        .border(Color.blue)
        .previewLayout(.fixed(width: 1000, height: 1000))
    }
}
{% endhighlight %}

If expanding behavior is observed in the first case and hugging behavior in the second for some direction, this means the composed view has chameleon behavior in this direction, since children ultimately determine its behavior.

If on the other hand the composed view ignores its children behavior for some direction, then it must either have expanding or hugging behavior in this direction. Simply apply the procedure described above for simple views to determine the intrinsic composed view behavior in this case.

TODO with VStack for example + screenshots

[^4]: These views have expanding, resp. hugging behavior, in all directions so that we can inspect horizontal and vertical behaviors at the same time.

### Probing Modifiers
{:.no_toc}

Modifiers return opaque composed views (as they are meant to augment the view they are applied on). Probing a modifier is therefore achieved in a similar way by using views with expanding and hugging behaviors as receivers:

{% highlight swift linenos %}
struct Modifier_Previews: PreviewProvider {
    static var previews: some View {
        Group {
            Color.red
            Text("Test")
        }
        .modifier(...)
        .border(Color.blue)
        .previewLayout(.fixed(width: 1000, height: 1000))
    }
}
{% endhighlight %}

TODO with VStack for example + screenshots + frame modifier

## Ambiguous Layouts

One of the biggest advantages of SwiftUI over Auto Layout is the fact that layouts can never break. Every seasoned UIKit developer knows that Auto Layout complains when layouts are over or underdetermined, yielding unpredictable results and logging warning messages to the console. This never occurs with SwiftUI.

This does not mean that ambiguities do not exist in SwiftUI layouts, though. When the SwiftUI layout engine encounters an ambiguity, it namely assigns a magical value of 10 to view dimensions it could not properly determine for some reason.

Such situations usually occur when a parent view wants to size itself based on the size of its children, but those children exhibit expanding behavior and thus want to match the parent size. This creates a usual chicken-and-egg problem which SwiftUI solves by replacing the child undetermined size with 10, as can be seen with the simple following code:

{% highlight swift linenos %}
struct Undetermined_Size_Previews: PreviewProvider {
    static var previews: some View {
        Color.red
            .fixedSize()
    }
}
{% endhighlight %}

Since `Color` has h-exp and v-exp behavior, the `fixedSize()` modifier cannot figure out the intrinsic size it needs to apply, and thus uses 10 as fallback in both directions. The layout above therefore displays a 10x10 red square but works fine, though of course the result obtained is not the expected one.

Therefore, when you see some size of 10 strangely appearing somewhere in your layout for unknown reasons, this usually means that some chicken-and-egg problem exists with the involved view and its parent. Nothing will break or throw an exception, but you should still have a look at why the problem exists in the first place and lift the associated ambiguity, for example by applying a modifier which will provide the child with a well-defined size.

## Sizing Behaviors in View Hierarchies

Layouts are created by assembling simple and composed views together. A typical layout therefore involves views with hugging, expanding and chameleon intrinsic sizing behaviors.

Associated with composed views only, the chameleon sizing behavior is only a placeholder for expanding or hugging behavior. When considering some layout, you therefore must first determine whether some chameleon behavior translates into expanding or hugging behavior first. This is key to understand how the layout will behave in practice.

Finding the true nature of a chameleon behavior is typically achieved in a top-bottom fashion, starting from a composed view with undetermined chameleon behavior, then recursively considering the behavior of its children. Once all chameleon behaviors are determined to be either expanding or hugging, you can proceed back to the composed view you started from in a bottom-top fashion to finally determine whether the original chameleon behavior you were interested in translates to expanding or hugging behavior.

This process might seem cumbersome but is thankfully theoretical in most cases. As SwiftUI views are usually small reusable units for which the behavior is known or can be quickly determined, the process above should in practice only involve a brief look at the children of some composed view to guess its overall behavior.

To speed up the process of identifying chameleon behaviors, it might still be useful to document custom views in your code so that their behavior can be quickly guessed from their documentation, for example:

{% highlight swift linenos %}
/// Intrinsic sizing behavior: h-exp, v-exp
struct CalendarView: View {
    // ...
}

/// Intrinsic sizing behavior: h-exp, v-hug
struct SegmentedControl: View {
    // ...
}

// Tag is a "Dual-Category View", see corresponding paragraph in the taxonomy section
struct Tag<Label: View>: View {
    /// Intrinsic sizing behavior: h-cha, v-cha
    init(@ViewBuilder label: @escaping () -> Label) {
        // ...
    }
    
    /// Intrinsic sizing behavior: h-hug, v-hug
    init(_ titleKey: LocalizedStringKey) where Label == Text {
        // ...
    }
}

extension View {
    /// Intrinsic sizing behavior: h-cha, v-cha
    func ornatePictureFrame() -> some View {
        // ...
    }
}
{% endhighlight %}

## Altering Sizing Behaviors

When the behavior of a SwiftUI view is not the one you want you cannot literally alter its properties directly, as views themselves are value types and thus immutable. Instead you wrap the view with unsatisfying behavior into another one to obtain the desired behavior. This is usually achieved by wrapping the view itself into another one, either using some public composed view (e.g. a stack) or by applying a modifier.

Composed views having chameleon intrinsic behavior depend on their children to determine whether their actual behavior is expanding or hugging. In addition to the approaches outlines above, such views therefore provide additional ways to influence their sizing behaviors, namely applying modifiers to their children, or simply adding expanding children like spacers.

## Layout Priorities

Layout priorities do not change the sizing behavior of a view. They merely are used by composed parent views to decide which child they should propose a size first.

For this reason this article will not further discuss layout priorities. You can read the [dedicated article from John Sundell](https://www.swiftbysundell.com/articles/swiftui-layout-system-guide-part-3/) to learn more about this topic.

## SwiftUI View Taxonomy

Now that we have explained the essential concepts behind SwiftUI layout system we can establish a list of the intrinsic sizing behaviors for most SwiftUI built-in views. These can be used as a reference when studying or building layouts.

Each view was probed using one of the procedures outlined in this document. Results were consolidated and presented in several tables, grouping views with common purpose.

Note that tables not only list behaviors of public view types like `Text`, but also of opaque types returned by modifiers like `Image/resizable()` or `View/frame(width:height:)`.

### Dual-Category Views
{:.no_toc}

Some views can either belong to simple views or composed views depending on how they are instantiated. You should be especially careful when using such views, as simply adding a trailing closure to them might change a simple view into a composed view with entirely different layout behaviors (usually switching from hugging to chameleon behavior).

Dual-category views include most notably `Label`, `Link`, `ProgressView`, `Slider`, `Stepper` and `Toggle`.

### Simple Building Blocks
{:.no_toc}

The following views are commonly used when building any kind of layout.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Color` | Simple | exp | exp |
| `Divider` | Simple | exp | hug |
| `Image` | Simple | hug | hug |
| `Image` from `resizable()` modifier | Simple | exp | exp |
| `SecureField` | Simple | exp | hug |
| `Text` | Simple | hug | hug |
| `TextEditor` | Simple | exp | exp |
| `TextField` | Simple | exp | hug |

### Buttons
{:.no_toc}

Button style can be controlled with the `Button/buttonStyle(_:)` modifier, resulting in different behaviors.

| Type | Category | Horizontal | Vertical | Remarks |
|:-- |:--:|:--:|:--:|:-- |
| `Button` (default) | Composed | cha | cha | No style or `DefaultButtonStyle`. Corresponds to `PlainButtonStyle` on iOS and to `BorderedButtonStyle` on tvOS |
| `Button` (plain) | Composed | cha | cha | `PlainButtonStyle`. No content insets are applied on tvOS |
| `Button` (bordered, tvOS only) | Composed | cha | cha | `BorderedButtonStyle`. Some content insets are applied |
| `Button` (card, tvOS only) | Composed | hug | hug | `CardButtonStyle`. ⚠️ This button calls `View/fixedSize(horizontal:vertical:)` on its content |
| `Button` (custom) | Composed | | | |

### Links
{:.no_toc}

Links can be created with a title `String` or with a custom label view.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Link` | Simple | hug | hug |
| `Link` with `Label: View` | Composed | cha | cha |

### Labels
{:.no_toc}

Labels can be created with a title `String` and an image / system image, or with two custom views for the title and the icon.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Label` | Simple | hug | hug |
| `Label` with `Title: View` and `Icon: View` | Composed | cha | cha |

### Stacks
{:.no_toc}

Stacks are essential components for horizontally and vertically arranging views.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `HStack` | Composed | cha | cha |
| `VStack` | Composed | cha | cha |
| `ZStack`| Composed | cha | cha |
| `LazyHStack` | Composed | hug | exp |
| `LazyVStack` | Composed | exp | hug |

Note that lazy stacks have very different sizing behaviors from their standard counterparts. You should therefore be especially careful when you switch a stack for its lazy counterpart, as this will change your sizing behavior and might require some layout adjustments.

### Grids
{:.no_toc}

- HGrid / VGrid
- LazyHGrid / LazyVGrid

### Sliders
{:.no_toc}

Sliders can be created with or without associated custom label view.

| Type | Category | Horizontal | Vertical | Remarks |
|:-- |:--:|:--:|:--:|:-- |
| `Slider` | Simple | exp | hug | |
| `Slider` with `Label: View` | Composed | exp | hug | Label used only for accessibility; does not participate in the layout |

### Progress Views
{:.no_toc}

Progress views can be created with or without associated custom label view. Their style can be controlled with the `View/progressViewStyle(_:)` modifier, resulting in different behaviors.

| Type | Category | Horizontal | Vertical | Remarks |
|:-- |:--:|:--:|:--:|:-- |
| `ProgresView` (linear) | Simple | exp | hug | No style, `DefaultProgressViewStyle` or `LinearProgressViewStyle` |
| `ProgresView` (linear) with `Label: View` | Composed | exp | hug | No style, `DefaultProgressViewStyle` or `LinearProgressViewStyle`. Label used only for accessibility; does not participate in the layout |
| `ProgresView` (circular) | Simple | hug | hug | `CircularProgressViewStyle` |
| `ProgresView` (circular) with `Label: View` | Composed | cha | cha | `CircularProgressViewStyle`. Label displayed underneath |
| `ProgressView` (custom) | | | | |
| `ProgressView` (custom) with `Label: View` | | | | |

### Steppers
{:.no_toc}

Steppers can be created with a title `String`, or with a custom view for the title.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Stepper` | Simple | exp | hug |
| `Stepper` with `Label: View` | Composed | exp | cha |

### Toggles
{:.no_toc}

Toggles can be created with a title `String`, or with a custom view for the title.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Toggle` | Simple | exp | hug |
| `Toggle` with `Label: View` | Composed | exp | cha |

### Pickers
{:.no_toc}

TODO

### Shapes
{:.no_toc}

Several built-in shapes are available. Custom shapes can be created by implementing the `Shape` protocol. All have expanding behaviors in all directions.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Capsule` | Simple | exp | exp |
| `Circle` | Simple | exp | exp |
| `Ellipse` | Simple | exp | exp |
| `Rectangle` | Simple | exp | exp |
| `RoundedRectangle` | Simple | exp | exp |
| `Shape` (custom) | Simple | exp | exp |

### Spacers
{:.no_toc}

Spacers are always flexible and work only within stacks. You can create a fixed size spacer with the `View/frame(width:height:)` modifier.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `Spacer` | Simple | exp | exp |

### Simple Frame
{:.no_toc}

The `View/frame(width:height:alignment:)` is used to constrain space provided to its receiver in any or all directions:

| `width` / `height` argument | Obtained behavior in the horizontal / vertical direction |
|:-- |:--:|:--:|
| Omitted | cha |
| Finite value | hug |

If an argument is omitted for some direction, the frame wrapper transparently adopts the same behavior as the receiver in this direction.

### Advanced Frame
{:.no_toc}

The `View/frame(minWidth:idealWidth:maxWidth:minWeight:idealHeight:maxHeight:alignment:)` modifier is used to constrain space or provide as much space as possible to its receiver in some or all directions:

| `maxWidth` / `maxHeight` argument | Obtained behavior in the horizontal / vertical direction |
|:--:|:--:|
| Omitted | cha |
| Finite value | hug |
| `.infinity` | exp |

If an argument is omitted for some direction, the frame wrapper transparently adopts the same behavior as the receiver in this direction.

Note that only maximum values affect the sizing behavior of the frame. Minimal and ideal arguments are only considered when the frame has hugging behavior (finite maximum value) to choose the best possible size for itself.

### Aspect Ratio
{:.no_toc}

A view can be forced to a given aspect ratio with the `View/aspectRatio(_:contentMode:)` modifier. This modifier does not change the sizing behavior of the receiver, but if the receiver is expanding in all directions it guarantees that it fit or fills the parent view while maintaining the desired aspect ratio, depending on the content mode specified.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `View/aspectRatio(_:contentMode:)` | Composed | cha | cha |

The aspect ratio is an optional parameter. If omitted the intrinsic aspect ratio of the receiver is used.

### Fixed Size
{:.no_toc}

A view can be forced to its intrinsic size with the `View/fixedSize(horizontal:vertical:)` modifier.

| Type | Category | Horizontal | Vertical |
|:-- |:--:|:--:|:--:|
| `View/fixedSize(horizontal:vertical:)` | Composed | hug | hug |

If an argument is omitted for some direction, the frame wrapper transparently adopts the same behavior as the receiver in this direction. As discussed in the _Ambiguous Layout_ section, if the intrinsic dimension of the receiver cannot be determined SwiftUI will replace it with the magic value 10.

### Border, Background and Overlay
{:.no_toc}

TODO

### Offset
{:.no_toc}

TODO

### Special Views
{:.no_toc}

The following are special invisible views 

| Type | Category | Horizontal | Vertical | Remarks |
|:-- |:--:|:--:|:--:|:-- |
| `GeometryReader` | Composed | cha | cha | The `geometryProxy` can be used for placing children precisely, otherwise they are put at the origin (top left) |
| `Group` | Composed | cha | cha | |


- GroupBox
- ForEach
- ScrollView
- List
- Menu

### UIViewRepresentable / UIViewControllerRepresentable
{:.no_toc}

Depend on the prorities defined on the view (e.g. lowest hugging)

## Card button style investigation
{:.no_toc}

Create a tvOS Card button with a color as only child -> you see that a 10 x 10 rectangle is displaye.

For other styles it works differently (color expands).

Conclusion: Button card calls fixedSize modifier on its content. Problems with ZStack for example, as size of a ZStack is determined by its largest child

Solution: Create card button class with (see Play) with chameleon behavior

## Layout Examples

-> Or maybe not: Armed with this knowledge, it suffices to have a look at other articles and to play with previews.

- SwiftUI tvOS button like in Play
- Vertical cell layout
- Horizontal cell layout

Cell layout:

Image with fixed ratio
Text area underneath (flexible)
Icons at the top right of the image
iOS and tvOS implementations