---
title: Dymanically create NSString containing emoji
---

A question from [https://stackoverflow.com/q/38125210](https://stackoverflow.com/q/38125210), authored by [Iulian Onofrei](https://stackoverflow.com/users/865175/iulian-onofrei)

> ### Dynamically create NSString with Unicode emoji
> 
> I have the string `@"Hi there! \U0001F603"`, which correctly shows the emoji like `Hi there! 😃` if I put it in a `UILabel`.
> 
> But I want to create it dynamically like `[NSString stringWithFormat:@"Hi there! \U0001F60%ld", (long)arc4random_uniform(10)]`, but it doesn't even compile.
> If I double the backslash, it shows the Unicode value literally like `Hi there! \U0001F605`.
> 
> How can I achieve this?
 
---

## [My reply](https://stackoverflow.com/a/38131780)

A step back, for a second: that number that you have, 1F6603<sub>16</sub>, is a Unicode _code point_, which, to try to put it as simply as possible, is the index of this emoji in the list of all Unicode items. That's not the same thing as the bytes that the computer actually handles, which are the "encoded value" (technically, the code _units_. 

When you write the _literal_ `@"\U0001F603"` in your code, the compiler does the encoding for you, writing the necessary bytes.* If you don't have the literal at compile time, you must do the encoding yourself. That is, you must transform the code point into a set of bytes that represent it. For example, in the UTF-16 encoding that `NSString` uses internally, your code point is represented by the bytes `ff fe 3d d8 03 de`.

You can't, at run time, modify that literal and end up with the correct bytes, because the compiler has already done its work and gone to bed.

(You can read in depth about this stuff and how it pertains to `NSString` in [an article by Ole Begemann at objc.io][0].)

Fortunately, one of the available encodings, UTF-32, represents code points directly: the value of the bytes is the same as the code point's. In other words, if you assign your code point number to a 32-bit unsigned integer, you've got proper UTF-32-encoded data.

That leads us to the process you need:

    // Encoded start point
    uint32_t base_point_UTF32 = 0x1F600;
    
    // Generate random point
    uint32_t offset = arc4random_uniform(10);
    uint32_t new_point = base_point_UTF32 + offset;
    
    // Read the four bytes into NSString, interpreted as UTF-32LE.
    // Intel machines and iOS on ARM are little endian; others byte swap/change 
    // encoding as necessary.
    NSString * emoji = [[NSString alloc] initWithBytes:&new_point
                                                length:4
                                              encoding:NSUTF32LittleEndianStringEncoding];
 
(N.B. that this may not work as expected for an arbitrary code point; not all code points are valid.)

---

*Note, it does the same thing for "normal" strings like `@"b"`, as well.

[0]:https://www.objc.io/issues/9-strings/unicode/
[MR]:http://stackoverflow.com/a/23147593/
