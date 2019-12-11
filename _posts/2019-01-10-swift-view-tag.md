---
title: Extend UIView with a strongly-typed tag
---

A question from [https://stackoverflow.com/q/54135116](https://stackoverflow.com/q/54135116), authored by [Chewie The Chorkie](https://stackoverflow.com/users/586006/chewie-the-chorkie)

> ### Using a Swift enum as view tag numbers without rawValue
> 
> I have an integer enum which I'd like to use for the `viewWithTag(_:)` number, but it gives me the error "Cannot convert value of type 'viewTags' to expected argument type 'Int'", even though both the enum and the tag number needed in `viewWithTag(_:)` is an `Int`.
> 
> This is pretty simple, and I can get it to work if I use the `rawValue` property, but that's more messy and cumbersome than I'd like.
> 
>     enum viewTags: Int {
>     	case rotateMirroredBtn
>     	case iPhone6SP_7P_8P
>     	case iPhoneX_Xs
>     	case iPhoneXs_Max
>     	case iPhone_Xr
>     	case unknown
>     }
>     
>     // error on if statement "Cannot convert value of type 'viewTags' to expected argument type 'Int'"
>     if let tmpButton = self.view.viewWithTag(viewTags.rotateMirroredBtn) as? UIButton { 
>     	tmpButton.removeFromSuperview()
>     }
  
---

## [My reply](https://stackoverflow.com/a/54135510)

You can easily add an extension on `UIView` to do the conversion for you. You just need to use a generic parameter to restrict the argument to something that you can get an `Int` from.

    extension UIView
    {
        /**
         Returns the view’s nearest descendant (including itself) whose `tag`
         matches the raw value of the given value, or `nil` if no subview
         has that tag.
         - parameter tag: A value that can be converted to an `Int`.
         */
        func firstView <Tag : RawRepresentable> (taggedBy tag: Tag) -> UIView?
            where Tag.RawValue == Int
        {
            let intValue = tag.rawValue
            return self.viewWithTag(intValue)
        }
    }

The constraint `T : RawRepresentable where T.RawValue == Int` can be fulfilled by your `Int`-backed enum.

An non-generic form is easy too: `func firstView(taggedBy viewTag: ViewTag) -> UIView?`

Bonus, you can also add a method to _apply_ the raw value of a "composed" value to the view's:

    func applyTag <Tag : RawRepresentable> (_ tag: Tag)
        where Tag.RawValue == Int
    {
        self.tag = tag.rawValue
    }

(Unfortunately there's no way to write that as a property, e.g. `var composedTag: Tag where Tag : RawRepresentable, Tag.RawValue == Int` because a computed property can't create its own generic context like the method can.)

---

<sub>Bonus bonus, if you want to "do like ObjC did", that's basically just this: [gist.github.com/woolsweater/8bfd52e7e725d82b53acb13f869d55c2](gist.github.com/woolsweater/8bfd52e7e725d82b53acb13f869d55c2) Unfortunately the compiler won't number them for you then.</sub>
