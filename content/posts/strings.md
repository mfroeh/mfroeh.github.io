+++
title = "On Strings in Rust"
date = "2026-07-22"
draft = false
tags = ['rust', 'strings']
+++

Strings in Rust are **not** simple, and perhaps not the easiest starting point when learning Rust. There is more to Rust strings than initially meets the eye, and when learning to understand them, you encounter a large variety of Rust-specific concepts that pervade the language as a whole. The concepts touched in this post are: slices (and thus dynamically sized types, `?Sized`, wide/fat pointers), iterators, deref coercion, dot operator semantics, operator overloading, and Unicode (not specific to Rust, but still useful to know).

Apart from a spellchecker no LLMs were used in the writing of this blog post, all mistakes are mine alone.

In this blog post, I want to fully demystify the following (admittedly contrived) but instructive block of code.
```rust
// 1. str vs. String
let hello: String = String::from("Hello");
let world: &'static str = "Wörld";
assert_eq!(world.len(), 6);

let mut char_constructed_string: String = String::new();
char_constructed_string.push('a');
let string_as_str: &str = char_constructed_string.as_str();

let hello_world: String = hello + " " + world;
assert_eq!(hello_world, "Hello Wörld");

// 2. [char] vs. String/str and dot operator semantics
let world_chars: [char; 5] = ['W', 'ö', 'r', 'l', 'd'];
assert_eq!(world.len(), world_chars.len()); // ↯ assertion failure
assert_eq!(size_of_val(world), size_of_val(&world_chars)); // ↯ assertion failure

let mut owned_world: String = String::from(world);
assert_eq!(owned_world.chars().count(), world_chars.len());
owned_world.make_ascii_uppercase();
assert_eq!(owned_world, "WÖRLD"); // ↯ assertion failure
assert_eq!(owned_world, "WöRLD");
```

## 1. str vs. String
```rust
let hello: String = String::from("Hello");
let world: &'static str = "Wörld";
assert_eq!(world.len(), 6);

let mut char_constructed_string: String = String::new();
char_constructed_string.push('a');
let string_as_str: &str = char_constructed_string.as_str();

let hello_world: String = hello + " " + world;
assert_eq!(hello_world, "Hello Wörld");
```

[`str`](https://doc.rust-lang.org/stable/std/primitive.str.html) should be thought of as a wrapper around `[u8]` (i.e. `pub struct str([u8])`, the precise definition is compiler defined, since `str` is a primitive type).\
[`String`](https://doc.rust-lang.org/stable/std/string/struct.String.html) is literally defined as `pub struct String { vec: Vec<u8> }` [here](https://doc.rust-lang.org/stable/src/alloc/string.rs.html#353) in the `alloc` crate (which is re-exported by the standard library).

These definitions already tell us that the characters of a string are stored as bytes. However, safe Rust guarantees an additional invariant on those bytes for both `String` and `str`: their sequence is UTF-8 encoded [Unicode](https://en.wikipedia.org/wiki/Unicode). In UTF-8, every code point (think character) is encoded using 1 to 4 bytes. For example: all ASCII code points, the umlaut 'ö,' and the segmented digit six '🯶' are encoded using 1, 2, and 4 bytes, respectively [^1]. As a result, the `len()` of both `String` and `str` is the number of bytes required to encode their contained code points: `assert_eq!(world.len(), 6)`. This implementation detail seeps through the entirety of the `String`/`str` API, as we will see later.

So, how do `str` and `String` differ?

### str
[`str`](https://doc.rust-lang.org/stable/std/primitive.str.html) is a slice type and as such is also called string slice. Slices `[T]` are an abstraction over a contiguous sequence of elements of type `T`. For example, a slice `[u8]` can be used to represent all the following arrays: `[1,2,3]`, `[137]`, `[]`. This abstraction is incredibly powerful, as it allows you to write code that works on any contiguous sequence of elements, instead of only specific array sizes, whilst not requiring heap allocations in your interface. There is one "problem" however: what is the `size_of::<[u8]>()` such a slice? 

Slices are [dynamically sized types](https://doc.rust-lang.org/reference/dynamically-sized-types.html) (DSTs), which are types whose size cannot be known at compile time. To still be able to work with them in a compiled language, they are to be explicitly placed behind indirection in the form of a pointer, whose size can be known at compile time (which is array decay in C/C++).

With slices, however, we also want to know how many elements they contain, such as to avoid out-of-bounds access. Thus, we use a special type of pointer that can store 8 auxiliary bytes of metadata and shove the length in there. These special pointer types are called wide (or fat) pointers and are syntactically equivalent to regular pointers (pointers without metadata). As opposed to regular pointers, which are ~~`size_of::<usize>()`~~ 8 (sue me) bytes wide, these pointers are then 16 bytes wide. You can verify this easily: `assert_eq!(size_of::<Box<[u8]>>(), 16)`.

Most commonly, slices are encountered behind shared or exclusive references, `&[T]` or `&mut [T]`, which, of course, are then also 16 bytes wide.

Now to tie back from slices to `str`, which we know is basically just `[u8]`: `assert_eq!(size_of::<&str>(), 16)` and `assert_eq!(size_of::<Box<str>>(), 16)` - nice.

As we can see in the example, the string literal `"Wörld"` has type `&str`, which is a shared reference into the `.rdata` section of your binary, where the compiler put the UTF-8 encoded bytes of the literal, which, as we now know, is a wide pointer (or wide shared reference, if you will). Since string literals are literally embedded into your binary, they live for the entirety of the duration of your program and thus trivially have `'static` lifetime. Due to the concrete string literal being pointed to by a shared reference `&str`, the `str` is immutable, so you cannot change the underlying bytes that it points to. If you have a `&mut str` (or `Box<str>`) however, you can change the underlying bytes, and yes, `&mut str` is an obtainable type, as we will see later.

Note that for the slice, it does not matter where the memory it points to lies, and in particular, a string slice does not imply the existence of a string literal. The `char_constructed_string` sees no string literal at any point during its lifetime, but we can still obtain a string slice `&str` from it using `String::as_str`.

The only guaranteed ways in which string slices differ from owned strings `String` are that you cannot change which memory location a slice spans [^2] and that string slices must exist behind a wide pointer, whilst `String` has a compile-time constant size.

### String
[`String`](https://doc.rust-lang.org/stable/std/string/struct.String.html) is a wrapper around a `Vec<u8>`. `Vec` and as such, `String` owns heap-allocated memory that can be grown and (rarely required) shrunk. If you own a `String` or have an exclusive reference to one `&mut String`, you can modify its individual elements in place as with `&mut str`, but you can also `String::remove(usize)` elements, and most importantly, `String::push(char)` and `String::push_str(&str)`. Unlike `str`, `String` is conceptually simple. It is just a `Vec<u8>` that (in safe Rust) upholds the invariant that its elements make up valid UTF-8 when concatenated. Use-case wise, you should use `String` for strings that you need to mutate at some point and `&str` if you know that the string will never change, since `&str` does not imply heap allocation, whilst `String` always does. You can, however, also easily create a `String` from a `&str` using `String::from(&str)`. 

As can be seen in the example, `String` can be concatenated with `&str` using the `+` operator. This works because `String` implements [`std::ops::Add<&str>`](https://doc.rust-lang.org/stable/std/string/struct.String.html#impl-Add%3C%26str%3E-for-String). Implementing traits is the only way in which you can do operator overloading in Rust. The `std::ops::Add<Rhs>` trait is defined as 
```rust
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```
Since this trait is generic over the `Rhs`, it can be implemented several times on a concrete type. It would make sense to implement it for every type that should be usable with your type when on the right-hand side of the `+` operator. String implements it only once for `&str`.
```rust
impl Add<&str> for String {
    type Output = String;
    fn add(mut self, other: &str) -> String {
        self.push_str(other);
        self
    }
}
```
Note that due to `Output = String`, we can chain applications of `+`. For the example, this means the following desugaring:
```rust
let hello_world: String = hello + " " + world;
// is desugared to
use std::ops::Add;
let hello_world: String = <String as Add<&str>>::add(<String as Add<&str>>::add(hello, " "), world);
```

There is one more thing happening in the example: in `assert_eq!(hello_world, "Hello Wörld");`, we are comparing a `String` typed variable with a `&str` typed variable. Internally, the `assert_eq!` declarative macro simply invokes the `==` operator. This operator can be overloaded by implementing the `std::cmp::PartialEq<Rhs>`. String [implements it](https://doc.rust-lang.org/stable/std/string/struct.String.html#impl-PartialEq%3C%26str%3E-for-String) with `Rhs = &str`, so this works.

## 2. [Char] vs. String/str and dot operator semantics
```rust
let world_chars: [char; 5] = ['W', 'ö', 'r', 'l', 'd'];
assert_eq!(world.len(), world_chars.len()); // ↯ assertion failure
assert_eq!(size_of_val(world), size_of_val(&world_chars)); // ↯ assertion failure

let mut owned_world: String = String::from(world);
assert_eq!(owned_world.chars().count(), world_chars.len());
owned_world.make_ascii_uppercase();
assert_eq!(owned_world, "WÖRLD"); // ↯ assertion failure
assert_eq!(owned_world, "WöRLD");
```
Rust has a primitive `char` type. A `char` is a UTF-8 encoded Unicode scalar value (which is a subset of the Unicode code points). Since Unicode scalar values can be up to 4 bytes large, `assert_eq!(size_of::<char>(), 4)`. This is also why the first two assertions of the example fail: `String`/`str` use the maximally efficient UTF-8 encoding of the string they represent, whilst `[char; 5]` is just a simple array, whose size is `5 * size_of::<char>()`.

To obtain an `Iterator<Item = char>` over the Unicode scalar values of a `str`, use the [`str::chars()`](https://doc.rust-lang.org/stable/std/primitive.str.html#method.chars) method. Wait, but in the example above, we are calling `chars()` on a variable of type `String`, but `String` has no method `chars`, so this should not compile? The code does compile, and whilst it may look like magic at first, the reason why this compiles uses two concepts in Rust that you are likely encountering many times daily, potentially without realizing it or whilst accepting it as magic: Deref coercion and dot operator semantics.

### Deref coercion
Deref coercion is a concept enabled by the [`std::ops::Deref`](https://doc.rust-lang.org/stable/std/ops/trait.Deref.html) trait, which is defined as follows:
```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```
Unlike the `std::ops::Add` trait we saw above, `Deref` is not generic and can thus only be implemented by a type. Deref has an associated type `Target` and a method `deref` to return a shared reference to the `Target` type. To understand the type "bound" `?Sized`, let's see how `String` implements `Deref`:
```rust
impl ops::Deref for String {
    type Target = str;
    fn deref(&self) -> &str {
        self.as_str()
    }
}
```
From "String vs. str" we know that `str` is a slice type and thus a DST - a type whose size cannot be known at compile time. Since most Rust types are not DSTs, Rust places an implicit bound of the `Sized` trait on any type parameter [^3]. The `Sized` trait is an auto trait that is compiler-implemented on all non-DST types and simply states that the size is a compile-time constant. The `Deref` trait, however, does at no point require a stack allocation of the `Target` type. It takes a shared reference to the implementer in `&self` and returns a shared reference to `Target`. Therefore, it uses the `?Sized` syntax for lifting the implicit, compiler-added requirement of `Target: Sized`. The `?Trait` syntax only works for `Sized`. This is the reason why we are even allowed to implement `Deref<Target = str>` on `String`.

The `<String as Deref>::deref` implementation then simply forwards to `String::as_str`, which just returns a shared reference to the slice of the vector's elements.

Deref coercion allows the compiler to implicitly convert (coerce) `&T` to `&U`, if `T` implements `Deref<Target = U>` [^4]. So a value of type `&String` can be treated as a value of type `&str` - nice. However, in our example above, we have `String` not `&String`, so deref coercion on its own does not suffice - yet!

### Dot operator semantics
In a method call expression `receiver.method(...)` where `receiver` has type `R`, the dot operator is the name for the mechanism that resolves the name `method` to a concrete method implementation. The compiler starts by checking if there is a method named `method` that can be called directly on a value of type `R`. It checks the methods directly implemented on `R` (inherent methods), and if there is no matching one, checks the methods of the in-scope traits that `R` implements. If there is none, it applies auto-(mut)-refing on `receiver`, and, in the same order, checks if there is a match for either `&R` or `&mut R`. If there is still none, it checks if `R` has a `Deref` and optionally a `DerefMut` implementation, and if it does, applies `<R as Deref>::deref(&R)` and (if it exists) `<R as DerefMut>::deref_mut(&mut R)` and repeats the same steps recursively (note that neither `Deref` nor `DerefMut` has to be in scope for this to work) [^5].

In our example, we start with `R = String`. There is no `chars` method on `String` or any of its traits, so the initial by-value check `String::chars(owned_world)`, as well as the auto-reffed calls on `&owned_world` and `&mut owned_world` cannot be resolved. `String` is `Deref<Target = str>`, so we apply `<String as Deref>::deref(&owned_world)` and check `&str`, which does have a matching function `str::chars`, and we are done. The `owned_world.chars()` call thus desugars to `str::chars(<String as Deref>::deref(&owned_world))`. In fact, due to Deref coercion, `str::chars(&owned_world)` also works.

Deref coercion and dot operator semantics are incredibly powerful concepts, without which Rust would be far less ergonomic. They are also not that magical or complicated; you just need to know of them.

The final example I want to discuss is given in `owned_world.make_ascii_uppercase();`. `str::make_ascii_uppercase` is a function defined on `&mut str`. In addition to `Deref`, `String` implements `DerefMut` which, as the name implies, simply returns `&mut str`. Note that `Deref` is a super-trait of `DerefMut`, so `DerefMut` simply re-uses the `Target` type from `Deref`, which ensures that `Deref` and `DerefMut` are always consistent in their `Target`. The method call thus desugars to `str::make_ascii_uppercase(<String as DerefMut>::deref_mut(&mut owned_world))`.

The potentially surprising thing here is that the resulting string is `"WöRLD"` (lower-case 'ö'), instead of `"WÖRLD"`. One might say that the reason is given in the method name, which is correct, but there is also a mechanical reason why this is the only possible outcome for an operation that modifies `&mut str` in place, which is given in the docs of [`String::to_uppercase`](https://doc.rust-lang.org/stable/std/string/struct.String.html#method.to_uppercase): in Unicode, a code point's upper-case value may take up more bytes than the lower-case value. For ASCII, we know that the lower- and upper-case value of any code point is 1 byte, but this does not hold for all characters representable using Unicode. For example: `assert_eq!('ᾨ'.to_uppercase().to_string(), "ὨΙ");`. Since we cannot change which memory location the slice spans, and we only have access to `&mut str <=> &mut [u8]`, we can generally not perform this operation.

## Other string types
Apart from `String` and `str`, there are a few other string or string adjacent types:

* [`std::ffi::OsString`](https://doc.rust-lang.org/stable/std/ffi/struct.OsString.html) and its slice counterpart [`std::ffi::OsStr`](https://doc.rust-lang.org/stable/std/ffi/struct.OsStr.html): types for converting between Rust's `String`/`str` and the OS-dependent string format, which may not be UTF-8 encoded Unicode. Example: used when reading filenames of a directory.
* [`std::ffi::CString`](https://doc.rust-lang.org/stable/std/ffi/struct.CString.html) and its slice counterpart [`std::ffi::CStr`](https://doc.rust-lang.org/stable/std/ffi/struct.CStr.html): types that represent C-compatible (null-terminated) strings. You will use this when calling or obtaining results from C code (e.g., libc). You can create a `&'static CStr` from a literal using `c"Hello World"`.
* [`std::bstr::ByteString`](https://doc.rust-lang.org/stable/std/bstr/struct.ByteString.html) and its slice counterpart [`std::bstr::ByteStr`](https://doc.rust-lang.org/stable/std/bstr/struct.ByteStr.html): types for strings that are not necessarily valid UTF-8. Currently nightly behind `#![feature(bstr)]`.
* [`std::ascii::Char`](https://doc.rust-lang.org/stable/core/ascii/enum.Char.html): a 1-byte wide `char` type for representing the ASCII subset of Unicode. Useful if you know that the strings you're dealing with are guaranteed ASCII. In that case, `[std::ascii::Char`] can be more safe/convenient to work with than dealing with the `str::as_bytes`. Currently nightly behind `#![feature(ascii_char)]`.

## Changelog
2026-07-23: Fixed mixup of `"WöRLD"` and `"WÖRLD"` in text. Clarified assertion failures in snippets.

--- 
[^1]: https://www.charset.org/utf-8

[^2]: You can create a new slice from an existing slice, e.g. by indexing it `let smaller_slice = slice[1..]`, but you cannot change the memory location that is spanned by the slice in place.

[^3]: There is one exception, which is the `Self` parameter of traits. The `Self` parameter refers to the concrete type implementing the trait, thus requiring `Self: Sized` would imply that the trait could only be implemented on `Sized` types, which is often not a sensible default. You can still explicitly require that only `Sized` types implement your trait by making `Sized` a super trait of your trait.

[^4]: Furthermore, if `T: Deref<Target = U>`, it enables the deref operator `*` to be used as `*T`, which yields a place expression of type `U` (given a `t: T`, `*t` desugars to `*Deref::deref(&t)`). This is particularly useful with smart pointer types such as `Box` or `Rc`.

[^5]: There is also a final unsizing step that can happen, which is not relevant to these examples, so I left it out. Check here for more details: https://rustc-dev-guide.rust-lang.org/hir-typeck/method-lookup.html
