---
layout: post
title: Yet another article about method swizzling
---

Many Objective-C developers disregard method swizzling, considering it a bad practice. I don't like method swizzling, I love it. Of course it is risky and can hurt you like a bullet. Carefully done, though, it makes it possible to fill annoying gaps in system frameworks which would be impossible to fill otherwise. From simply providing a convenient way to [track the parent popover controller of a view controller](https://github.com/defagos/CoconutKit/blob/03c18648ce76a822e519e34b0fea6f66b6eb370e/CoconutKit/Sources/ViewControllers/UIPopoverController+HLSExtensions.m#L16-L78) to implementing [Cocoa-like bindings on iOS](https://github.com/defagos/CoconutKit/blob/03c18648ce76a822e519e34b0fea6f66b6eb370e/CoconutKit/Sources/Bindings/UIView+HLSViewBinding.m#L239-L257), method swizzling has always been an invaluable tool to me.

Implementing swizzling correctly is not easy, though, probably because it looks straightforward at first (all is needed is a few Objective-C runtime function calls, after all). Though the Web is [crawling with articles](https://www.google.ch/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=objective-c%20swizzling%20right) about the right way to swizzle a method, I sadly found issues with all of them.

The implementation discussed in this article may have issues of its own, but I do hope sharing it will help improve existing implementations, as well as mine. For this reason, do not regard these few words as a _Done right_ article, only as a small step towards hopefully better swizzling implementations. There is a correct way to swizzle methods, but it probably still remains to be found.

# Function prototype

Since the Objective-C runtime is a series of functions, I decided to implement swizzling as a function as well. Most existing implementations, for example the well-respected [JRSwizzle](https://github.com/rentzsch/jrswizzle), exchange `IMP`s associated with two selectors. Ultimately, though, method swizzling is about changing, not exchanging, which is why I prefer a function expecting an original `SEL` and an `IMP` arguments:

{% highlight objective-c %}
IMP class_swizzleSelector(Class clazz, SEL selector, IMP newImplementation);
{% endhighlight %} 

instead of two `SEL` arguments:

{% highlight objective-c %}
IMP class_swizzleSelectorWithSelector(Class clazz, SEL selector, SEL swizzlingSelector); 
{% endhighlight %}

This also avoid potential clashes if your swizzling selector name convention is the same as the one used elsewhere, especially when dealing with 3rd party code. Don't be too optimistic, accidental overriding due to [bad conventions](https://github.com/search?l=objective-c&q=%22%28void%29commonInit%22&ref=searchresults&type=Code&utf8=%E2%9C%93) can happen all the time.

The function returns the original implementation, which must be properly cast and called from within the swizzling method implementation, so that the original behavior is preserved. If the method to swizzle is not implemented, the function does nothing.

# Issues in class hierarchies

When swizzling methods in class hierarchies, we must take extra care when the method we swizzle is not implemented by the class on which it is swizzled, but is implemented by one of its parents. For example, the `-awakeFromNib` method, declared and implemented at the `NSObject` level, is neither implemented by the `UIView` nor by the `UILabel` subclasses. When calling this method on an instance of any of theses classes, it is therefore the `NSObject` implementation which gets called:

![Standard hierarchy](/images/standard_hierarchy.png)

If we naively swizzle the `-awakeFromNib` method both at the `UIView` and `UILabel` levels, we get the following result:

![Standard hierarchy, swizzled](/images/standard_hierarchy_swizzled.png)

As we see, when `-[UILabel awakeFromNib]` is now called, the swizzled `UIView `implementation does not get called, which is not what is expected from proper swizzling.

The situation would be completely different if the `-awakeFromNib` method was implemented on `UIView` and `UILabel`. If this was the case, and if each implementation properly called the `super` method counterpart first, we would namely obtain:

![Tweaked hierarchy](/images/tweaked_hierarchy.png)

and, after swizzling:

![Tweaked hierarchy, swizzled](/images/tweaked_hierarchy_swizzled.png)

No swizzling implementation I encountered correctly deals with this issue, [not even JRSwizzle](https://github.com/rentzsch/jrswizzle/issues/4). As should be clear from the last picture above, the solution to this problem is to ensure a method is always implemented by a class before swizzling it. If this is not the case, an implementation must be injected first, simply calling the super method counterpart. This way, all implementations will correctly be called after swizzling.

# Implementation

I therefore implemented instance method swizzling as follows:

{% highlight objective-c %}
#import <objc/runtime.h>
#import <objc/message.h>

IMP class_swizzleSelector(Class clazz, SEL selector, IMP newImplementation)
{
    // If the method does not exist for this class, do nothing
    Method method = class_getInstanceMethod(clazz, selector);
    if (! method) {
        return NULL;
    }
    
    // Make sure the class implements the method. If this is not the case, inject an implementation, calling 'super'
    const char *types = method_getTypeEncoding(method);
    class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
        struct objc_super super = {
            .receiver = self,
            .super_class = class_getSuperclass(clazz)
        };
        
        id (*objc_msgSendSuper_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper;
        return objc_msgSendSuper_typed(&super, selector, argp);
    }), types);
    
    // Can now safely swizzle
    return class_replaceMethod(clazz, selector, newImplementation, types);
}
{% endhighlight %}
For class method swizzling, it suffices to call the above function on a metaclass:

{% highlight objective-c %}
IMP class_swizzleClassSelector(Class clazz, SEL selector, IMP newImplementation)
{
    return class_swizzleSelector(objc_getMetaClass(class_getName(clazz)), selector, newImplementation);
}
{% endhighlight %}

The `imp_implementationWithBlock` function is used as a trampoline to accomodate any kind of method prototype through a variable argument list `va_list`. The `super` method call is made by properly casting `objc_msgSendSuper`, available from `<objc/message.h>`. In order to prevent ARC from inserting incorrect memory management calls, the `self` parameter of the implementation block has been marked with `__unsafe_unretained`.

This implementation is available from [my CoconutKit library](https://github.com/defagos/CoconutKit) with other runtime additions.

# Use

Define a static C-function for the new implementation and call `class_swizzleSelector` or `class_swizzlClassSelector` to set it as new implementation. Save the original implementation into a function pointer matching the function signature, and make sure the new implementation calls it somehow:

{% highlight objective-c %}
static id (*initWithFrame)(id, SEL, CGRect) = NULL;
static void (*awakeFromNib)(id, SEL) = NULL;
static void (*dealloc)(__unsafe_unretained id, SEL) = NULL;

static id swizzle_initWithFrame(UILabel *self, SEL _cmd, CGRect frame)
{
    if ((self = initWithFrame(self, _cmd, frame))) {
        // ...
    }
    return self;
}

static void swizzle_awakeFromNib(UILabel *self, SEL _cmd)
{
    awakeFromNib(self, _cmd);
    
    // ...
}

static void swizzle_dealloc(__unsafe_unretained UILabel *self, SEL _cmd)
{
    // ...
    
    dealloc(self, _cmd);
}

@implementation UILabel (SwizzlingExamples)

+ (void)load
{
    initWithFrame = (typeof(initWithFrame))class_swizzleSelector(self, @selector(initWithFrame:), (IMP)swizzle_initWithFrame);
    awakeFromNib = (typeof(awakeFromNib))class_swizzleSelector(self, @selector(awakeFromNib), (IMP)swizzle_awakeFromNib);
    dealloc = (typeof(dealloc))class_swizzleSelector(self, sel_getUid("dealloc"), (IMP)swizzle_dealloc);
}

@end
{% endhighlight %} 

Note that I added an extra `__unsafe_unretained` specifier to the `swizzle_dealloc` prototype to ensure ARC does not insert additional memory management calls. I also cheated by getting the `dealloc` selector with `sel_getUid`, since `@selector(dealloc)` cannot be used with ARC.
