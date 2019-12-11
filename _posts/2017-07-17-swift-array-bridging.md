---
title: How does Swift Array bridge to NSArray?
---

A question from [https://stackoverflow.com/q/45116205](https://stackoverflow.com/q/45116205), authored by [Mo Abdul-Hameed](https://stackoverflow.com/users/4763963/mo-abdul-hameed)

> ### Swift's Array type is bridged to Foundation's NSArray class, how?
> 
> In [Swift Language Guide][1] we read the following:
> 
> > Swift’s Array type is bridged to Foundation’s NSArray class.
> 
> How can Swift's `Array` be bridged to Foundation's `NSArray` class when the first is a *value type* and the latter is a *reference type*? Doesn't bridging mean having an interface in a language to use a code in a different language?
> 
> 
>   [1]: https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/CollectionTypes.html#//apple_ref/doc/uid/TP40014097-CH8-ID107

---

## [My reply](https://stackoverflow.com/a/45144463/603977)

The value/reference distinction here is a bit of a red herring.

Bridging is (maybe surprisingly) straightforward. There's an internal protocol, [`_ObjectiveCBridgeable`][ObjCBridge], that describes a type which can be cast between an ObjC and a Swift type. The compiler replaces, e.g., your `mySwiftArray as NSArray` with a call to `_bridgeToObjectiveC()`.

You can see [`Array`'s conformance to the protocol][Array] in Foundation. It's simple Swift code: each method just constructs an instance of the appropriate bridged type.

So there's not really any relation to the fact that the native Swift `Array` is a value type.

As for that piece; while externally a "value", `Swift.Array` actually has an internal _pointer_ to its own storage. If you think about it for a second, this is the only sensible way to make it work. You don't want to be moving the 101 things in an array every time you assign it to a new variable. Just a nice quick copy of a pointer. (Of course you need [to do the copy if you want to _change_ something][cow], but it's delayed as long as possible.)

You can see basically the same behavior in a C struct with a field that's a reference to some allocated memory:


    typedef struct _Array {
        void * payload;
    } Array;

    Array c;
    c.payload = malloc( /* Whatever */ );
    Array d = c;

Assigning to `d` makes a copy of the _pointer_ to the storage, but there's only one chunk of allocated memory, which hasn't moved or been copied. (And to extend this backwards, you can "bridge" this to `NSArray` in the same way `Swift.Array` does: by providing an appropriate function that does the transformation.)

[cow]:http://chris.eidhof.nl/post/struct-semantics-in-swift/
[ObjCBridge]:https://github.com/apple/swift/blob/ca5c65f93ca8f89ccb51fac41a225081490d5f23/stdlib/public/core/BridgeObjectiveC.swift
[Array]:https://github.com/apple/swift-corelibs-foundation/blob/master/Foundation/Array.swift
