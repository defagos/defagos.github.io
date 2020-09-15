---
layout: post
title: Building a Collection For SwiftUI - Introduction
---

Since its introduction in iOS 6 `UICollectionView` has been an essential tool in the Apple developer toolbox. At the time of this writing (iOS 14 GM), SwiftUI still does not provide any kind of native collection, though. In parallel, UIKit collection view has received some of its most important enhancements in iOS 13, namely diffable data sources and compositional layouts, making it more appealing than ever.

When evaluating SwiftUI as a viable option for porting our [iOS apps](https://github.com/SRGSSR/playsrg-apple) to tvOS, one of the essential requirements to satisfy was to ensure grid layouts, used throughout our implementation, would be easy to manage in a SwiftUI port. I initially and naively thought so, but I didn't quite imagine where my investigations would lead me in the end.

SwiftUI still being a nascent framework I thought sharing my results could be helpful to other developers who might be considering SwiftUI for their own apps. Since the material to be covered is larger than initially expected, I opted out for a series of articles requiring some prior SwiftUI and UIKit knowledge, but which I tried to keep accessible for newcomers as well:

- [Part 1: Flirting with SwiftUI Collections](/swiftui_collection_part1): A mild introduction into the world of collection aleternatives in SwiftUI (and associated delusions).
- [Part 2: A SwiftUI Collection Prototype](/swiftui_collection_part2): Where we build a first working prototype of a SwiftUI collection from the ground up.
- [Part 3: Focus Management and Polishing](/swiftui_collection_part3): Where we solve a few issues of the prototype implementation and implement focus management for tvOS.
