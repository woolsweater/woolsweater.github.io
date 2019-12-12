---
title: How ObjC Blocks capture values
permalink: /soa/:title
slug: objc-block-capture
---

A question from [https://stackoverflow.com/questions/17813870](https://stackoverflow.com/questions/17813870), authored by [iCanObjSeeSharp](https://stackoverflow.com/users/2570370/icanobjseesharp)

> ### How does a Block capture the variables outside of its enclosing scope?
> 
> I know that an Objective-C Block can capture and set the value of variables outside of its enclosing scope. How does it do that?
 
---

## [My reply](https://stackoverflow.com/a/17819142)

It's actually fairly straightforward and described in Clang's Block Implementation Spec, in the ["Imported Variables"][1] section.

When the compiler encounters a Block like:

    ^{ if( numBalloons > numClowns) abort(); }

it creates a literal structure that includes -- among other things -- two elements that are important here. There's a function pointer to the executable code in the Block, and a `const` field for each variable that's referred to inside the Block. Something like this:

    struct __block_literal_1 {
        /* other fields */
        void (*invoke)(struct __block_literal_1 *);
        /* ... */
        const int numBalloons;
        const int numClowns;
    };

Notice that the `invoke` function will take a pointer to a struct of the kind that's being defined right here; that is, the Block passes itself in when executing its code. Thus, the code gets access to the members of the structure.

Right after the declaration, the compiler creates a definition of the Block, which simply uses the referenced variables to initialize the correct fields in the `struct`:

    struct __block_literal_1 __block_literal_1 = {
        /* Other fields */
        __block_invoke_2,  /* This function was also created by the compiler. */
        /* ... */
        numBalloons,  /* These two are the exact same variables as */ 
        numClowns     /* those referred to in the Block literal that you wrote. *
     };

Then, inside the `invoke` function, references to the captured variables are made like any other member of a struct, `the_block->numBalloons`.

The situation for object-type variables is a little more complicated, but the same principle applies.


[1]:http://clang.llvm.org/docs/Block-ABI-Apple.html#imported-variables