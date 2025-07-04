+++
title = "\"German string\" optimizations in Spellbook"
date = 2024-11-03
taxonomies.tags = ["Rust", "optimization", "data structures"]

[extra]
repo_view = true
+++

# Spellbook

[Spellbook](https://github.com/helix-editor/spellbook) is a Rust spell-checking library I've written the style of [Hunspell](https://github.com/hunspell/hunspell) to bring spell checking to the [Helix editor](https://github.com/helix-editor/helix). It's more-or-less a Rust rewrite of [Nuspell](https://github.com/nuspell/nuspell), which itself is more-or-less a rewrite of Hunspell. Spellbook has a pretty slim interface: you can instantiate a dictionary from Hunspell dictionary files and use it to check words. For a small example of how you might use Spellbook:

```rust
fn main() {
    // Dictionary files can be sourced from
    // <https://github.com/LibreOffice/dictionaries>
    let aff = std::fs::read_to_string("en_US.aff").unwrap();
    let dic = std::fs::read_to_string("en_US.dic").unwrap();
    let dict = spellbook::Dictionary::new(&aff, &dic).unwrap();

    let word = std::env::args().nth(1).expect("expected a word to check");

    if dict.check(&word) {
        println!("{word:?} is in the dictionary.");
    } else {
        println!("{word:?} is NOT in the dictionary.");
        std::process::exit(1);
    }
}
```

In this post we'll be looking at the string representation used in Spellbook and aiming to optimize it to save memory.

# Strings in Spellbook

How Spellbook works exactly is beyond the scope of this post, so this section gives a simplified overview and deals with simplified types. If you're interested in more details, check out the [Spellbook README](https://github.com/helix-editor/spellbook/blob/master/README.md#how-does-it-work) or [@zverok's Rebuilding the Spellchecker](https://zverok.space/spellchecker.html) blog post and the [Spellbook internals document](https://github.com/helix-editor/spellbook/blob/master/docs/internals.md).

A central part of the procedure to check a word is to look up word(s) in a hash table. This lookup table contains an entry for each "stem" in the dictionary. You might imagine that the `Dictionary` type is a wrapper around a `HashSet<String>`. This is correct in essence but Hunspell-like checkers don't store every possible word in memory. Instead there is some "compression." For an example from the `en_US` (American English) dictionary, the lookup table in Spellbook associates a stem "adventure" with a set of flags like 'D', 'R' and 'S'. The flags correspond to rules defined for the dictionary allowing transformations like prefixes and suffixes. 'D' for example allows adding the "d" (or "ed" or "ied", depending on the stem) suffix, producing "adventured." 'R' allows "adventurer" and 'S' allows "adventures." So we can imagine that the lookup table has a type similar to `HashMap<String, HashSet<Flag>>`.

Despite the "compression" that prefixes and suffixes enable, the lookup table contains many entries. The exact number varies with which dictionary files you use as input but American English contains around 50,000 stems, and it's a relatively slim dictionary. Others contain hundreds of thousands or even millions of stems, so it's worth trying to optimize the space we take for each stem.

Good optimizations come from good observations so let's list out some properties of these strings:

* Once inserted into the lookup table these strings are never modified.
* These strings have a small maximum size. Spellbook refuses to check words over 360 bytes long (in UTF-8 representation) so there's no point in storing words over 360 bytes in the lookup table.
* Stems correspond to words so they're typically shorter rather than longer.

# Strings in Rust

Let's take a bit of a detour to talk about how strings are represented in Rust. For starters there's the `String` type. `String`s are quite flexible: they can be modified, resized and have a large maximum size. As for how they are represented, the Rust docs say:

> A `String` is made up of three components: a pointer to some bytes, a length, and a capacity.

Simplifying a bit here, we can imagine a String looks like this:

```rust
struct String {
    pointer: NonNull<u8>,
    length: usize,
    capacity: usize,
}
```

# `Box<str>` and fat pointers

The first thing that comes to mind is that storing `length` and `capacity` is redundant for our use-case. In our lookup table the strings are never modified so there's no need to store any extra information that would allow us to resize the string. A non-resizable string can be written with the `Box<str>` type. `Box<str>` is the owned version of a `&str`.

`&str` and slices (`&[T]`) have an interesting representation and learning about them is a good way to dig into "fat pointers" in Rust. A `&str` (or equivalently, `&[u8]`) is a fat pointer - a pointer to some bytes plus some metadata. For `&[T]` the metadata is the length of the slice. Using a fat pointer makes string (`&str`) and other slices nice to work with - you can subslice and read the length of a string slice cheaply and ergonomically. `Box<str>` and `Box<[T]>` are laid out the same way.

You can imagine that these fat pointers are basically a tuple `(*const T, usize)`. This takes 2 `usize`s worth of memory to represent: one `usize` for the actual pointer ("thin pointer") and one for the metadata. What exactly is a `usize` though? Quoting the Rust docs again:

> The size of \[`usize`\] is how many bytes it takes to reference any location in memory. For example, on a 32 bit target, this is 4 bytes and on a 64 bit target, this is 8 bytes.

So `usize` is an unsigned integer type of the same size as a "thin pointer": a pointer with no metadata, like `*const T`/`*mut T` or equivalently `NonNull<T>`. For simplicity we'll talk only about 64 bit targets for the rest of the post and assume that `size_of::<usize>() == 8`.

By switching the stem type to `Box<str>` we save 8 bytes per stem from not tracking `capacity`, taking advantage of our observation that strings are not modified. Nice! But there's still room for improvement from our other observations.

# The road to "German strings"

The other observations are about the length of each string. They're short. If the `length` field is a `usize` that means your strings can be at most 2^64 bytes long, and _wow that is long_! Our strings will never be longer than 360 bytes so of the 64 bits we use to represent the length we'll only ever use 9 (2^9 = 512). That's quite a few bits wasted. If we used a `u16` to store the length instead we'd save 6 bytes. What should we do with those 6 bytes we've saved?

This is where "German strings" come in. "German strings" or "German-style strings" or "Umbra strings" (all the same thing) are described very well in a post from CedarDB: [Why German Strings are Everywhere](https://cedardb.com/blog/german_strings/). The idea is to use a integer type smaller than `usize` for the length (`u32` in their case) and repurpose the remaining bytes to store a prefix of the data. We can store a few more bytes in the "prefix" section since we're using a `u16` for length, so our type would look like this in memory:

```rust
#[repr(C)]
struct UmbraString {
    len: u16, // takes 2 bytes
    prefix: [u8; 6], // takes 6 bytes
    pointer: NonNull<u8>, // this takes `usize` (8 bytes)
}
```

```
+-------+-----------------------+-------------------------------+
+  len  +        prefix         +           pointer             +
+-------+-----------------------+-------------------------------+
   u16           6x u8                       8x u8
```

Umbra and CedarDB like this prefix because it can be used to cheaply compute whether two of these `UmbraString`s are (not) equal - the `Eq` trait in Rust.

Consider a very short string like "hi!". In memory that would look like so:

```
+-------+-----------------------+-------------------------------+
+ 3u16  + h   i   !   .   .   . +         pointer (?)           +
+-------+-----------------------+-------------------------------+
```

And what's the pointer pointing to? Nothing I guess. We already stored the full string right in the struct "inline." So there's no need to allocate memory and point to it.

In fact for medium-long strings that can fit in the prefix bytes plus the pointer bytes, we can eliminate the pointer part altogether. This is a **S**hort **S**tring **O**ptimization (SSO): when the string is short enough, we can store it directly in our `UmbraString` struct and avoid allocating a buffer. We can store 6 bytes in the prefix and another 8 in the suffix area for a total of 14 bytes inline. For an ASCII string, that's up to 14 characters we can represent without allocating. Very nice!

```
+-------+-----------------------+-------------------------------+
+ 12u16 + h   e   l   l   o  _  + w   o   r   l   d   !   .   . +
+-------+-----------------------+-------------------------------+
   len           prefix                     suffix
```

This either-or type would look like so, using a `union`:

```rust
#[repr(C)]
struct UmbraString {
    len: u16,
    prefix: [u8; 6],
    trailing: Trailing
}

#[repr(C)]
union Trailing {
    suffix: [u8; 8],
    // ManuallyDrop is necessary since we only want
    // to deallocate the buffer if we're using the
    // "long" variant of this union.
    ptr: ManuallyDrop<NonNull<u8>>,
}
```

How do we know which member of the `union` our `UmbraString` is? Just look at the `len` field: if it's 14 or less then we're using the "short" variant - everything inline. If it's 15 or greater then the string is allocated and pointed to.

# Memory savings

Why is this layout so attractive?

This representation is no more expensive than a `Box<str>` in terms of memory consumption. `size_of::<Box<str>>()` is `16` - 16 bytes. (Note that `size_of` is counting the size of the type, not the size of the allocation the pointer is pointing to.) `size_of::<UmbraString>()` is also `16`. The difference is that any non-empty `Box<str>` will allocate. A short string like "hi!" allocates 3 bytes somewhere on the heap for a total of 19 bytes. `UmbraString` does not: it's still 16 bytes. For a medium string like "hello_world!" `Box<str>` will allocate those 12 bytes on the heap for a total cost of 28 bytes. The equivalent `UmbraString` is still a total of 16 bytes. For long strings like `"a".repeat(50)`, `Box<str>` will allocate the 50 bytes for a total cost of 66 bytes. In the worst case (long strings) `UmbraString` **is no worse**: it also takes exactly 66 bytes.

Umbra strings are attractive here because they don't have a memory cost: we would be paying the 16 bytes of a `Box<str>` anyways and wasting the 6 bytes from the length `usize`. Any time we use the inline variant of `UmbraString` we save memory.

You might also think `UmbraString` is faster to work with if you commonly have short strings because you don't need to follow a pointer to compare data. We'll see in the benchmarks that `UmbraString` is not much different in terms of speed. We need an extra comparison operation to figure out if we're using a short or long variant after all.

# Theory into practice: let's build `UmbraString`

This is basically the same snippet as above. We'll define some constants for the lengths of each segment and some basic helpers.

```rust
use core::mem::{size_of, ManuallyDrop};
use core::ptr::NonNull;

// 6 on 64 bit machines
const PREFIX_LEN: usize = size_of::<usize>() - size_of::<u16>();
// 8 on 64 bit machines
const SUFFIX_LEN: usize = size_of::<usize>();
// We can fit 14 bytes inline, nice!
const INLINE_LEN: u16 = (PREFIX_LEN + SUFFIX_LEN) as u16;

#[repr(C)]
pub struct UmbraString {
    len: u16,
    prefix: [u8; PREFIX_LEN],
    trailing: Trailing,
}

#[repr(C)]
union Trailing {
    suffix: [u8; SUFFIX_LEN],
    ptr: ManuallyDrop<NonNull<u8>>,
}

impl UmbraString {
    pub fn len(&self) -> usize {
        self.len as usize
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
}
```

## `Default`

The empty string is easy to represent: the length is `0` so it belongs as the inline variant. We'll set everything to zero - we won't access those bytes so it doesn't really matter what they're set to, but this seems like a reasonable default.

```rust
impl Default for UmbraString {
    fn default() -> Self {
        Self {
            len: 0,
            prefix: [0; PREFIX_LEN],
            trailing: Trailing {
                suffix: [0; SUFFIX_LEN]
            }
        }
    }
}
```

## Allocating

Let's define some helper functions for actually allocating the data. The allocation helpers are only used when working with the long variant. A `&str` is a `&[u8]` that is valid UTF-8 so we'll be working in terms of `*mut u8`/`*const u8` thin pointers.

```rust
use alloc::alloc;
use core::ptr::{self, NonNull};

fn copy_slice(src: &[u8]) -> NonNull<u8> {
    let layout = layout(src.len());
    let nullable = unsafe { alloc::alloc(layout) };
    let ptr = match NonNull::new(nullable) {
        Some(ptr) => ptr.cast(),
        None => alloc::handle_alloc_error(layout),
    };
    unsafe {
        ptr::copy_nonoverlapping(src.as_ptr(), ptr.as_ptr(), source.len());
    }
    ptr
}

fn layout(len: usize) -> alloc::Layout {
    alloc::Layout::array::<u8>(len)
        .expect("a valid layout for an array")
        .pad_to_align()
}
```

`copy_slice` allocates an array of bytes on the heap and then copies the source byte slice into our new array, and returns the pointer.

## Instantiation

To create an `UmbraString` we'll take an existing `&str` as input. This operation could possibly fail if the input string is too long. Let's ignore that for now and just `assert!` that the string is not too long:

```rust
impl From<str> for UmbraString {
    fn from(src: &src) -> Self {
        assert!(src.len() <= u16::MAX as usize);
        let len = src.len();

        let mut prefix = [0; PREFIX_LEN];
        let trailing = if len as u16 <= INLINE_LEN {
            let suffix = [0; SUFFIX_LEN];
            if len <= PREFIX_LEN {
                prefix[..len].copy_from_slice(source);
            } else {
                prefix.copy_from_slice(&source[..PREFIX_LEN]);
                suffix[..len - PREFIX_LEN].copy_from_slice(&source[PREFIX_LEN..]);
            }

            Trailing { suffix }
        } else {
            let ptr = copy_slice(source);
            Trailing { ptr: ManuallyDrop::new(ptr) }
        }

        Self {
            len: len as u16,
            prefix,
            trailing
        }
    }
}
```

For the short variant (`src.len() as u16 <= INLINE_LEN`) we copy from the source byte slice into however much of the prefix and suffix slices we can fill and leave the rest as `0`s. (Note that `0` is a valid representation in UTF-8. See the section below on `FlagSet`s for more discussion on why this is important.)

For the long variant we'll use our `copy_slice` helper from above to allocate a new byte array pointer.

## Reconstructing a byte slice

Did you notice in our `copy_slice` helper function above that we copy the entire slice into a newly allocated array buffer instead of the part after the prefix? We copied `source` instead of `&source[PREFIX_LEN..]`. You might think that we could save some space by only storing the remaining bytes after the prefix - and we could - but that would prevent us from recreating a `&[u8]` or `&str` from an `UmbraString`. Slices are **contiguous** memory chunks - array layouts in memory. We can't create a slice that starts in the `prefix` field and then continues by following a pointer. All of the data needs to be in one place.

With that in mind, let's add a function to get our bytes back:

```rust
use core::{ptr, slice};

impl UmbraString {
    pub fn as_slice(&self) -> &[u8] {
        let ptr = if self.len <= INLINE_LEN {
            let ptr = ptr::from_ref(self);
            unsafe { ptr::addr_of!((*ptr).prefix) }.cast()
        } else {
            unsafe { self.trailing.ptr }.as_ptr()
        };

        unsafe { slice::from_raw_parts(ptr, self.len()) }
    }

    pub fn as_bytes(&self) -> &[u8] {
        self.as_slice()
    }

    pub fn as_str(&self) -> &str {
        unsafe { core::str::from_utf8_unchecked(self.as_slice()) }
    }
}
```

For inline Umbra strings our slice starts at the prefix field and ends either in the prefix field's array or in the suffix field's array depending on the length. The `#[repr(C)]` annotation on `UmbraString` and `Trailing` enforces that when represented in memory at runtime, the fields are in the same order as we define them, so we can safely assume that `prefix` comes before `suffix` and there's no space between. We can safely treat them as contiguous memory. For allocated strings we reconstruct the slice directly from our allocated buffer's pointer.

Remember earlier when we said that slices were basically `(*const T, usize)`? That's what we give to `slice::from_raw_parts` - a pointer to an array layout in memory and a length - and we get a fat pointer.

## Clone

Cloning the string is similar to how we initially created one from a `&str`.

```rust
impl Clone for UmbraString {
    fn clone(&self) -> Self {
        let trailing = if self.len <= INLINE_LEN {
            let suffix = unsafe { self.trailing.suffix };
            Trailing { suffix }
        } else {
            let ptr = copy_slice(self.as_slice());
            Trailing { ptr: ManuallyDrop::new(ptr) }
        };

        Self {
            len: self.len,
            prefix: self.prefix,
            trailing,
        }
    }
}
```

The `len` and `prefix` fields are copied. For the inline version we copy the `suffix` array too, and for the allocated version we create a new allocation and copy `self`'s buffer.

Another nice property of this type you might notice here: for strings short enough to be inlined, `Clone` is actually a `Copy` - no allocation required.

## Drop

Now on to `Drop`. We need to deallocate our allocated buffer for the long variant. For the short variant we do nothing: `Copy` types are cleaned up without any mention in `Drop`.

```rust
impl Drop for UmbraString {
    fn drop(&self) {
        if self.len > INLINE_LEN {
            let layout = layout(self.len());
            let ptr = unsafe { self.trailing.ptr }.as_ptr();
            unsafe { alloc::dealloc(ptr.cast(), layout);
        }
    }
}
```

## `Eq`

As the CedarDB article notes, we can optimize the comparison of Umbra strings. To do that we cast the `len` and `prefix` chunks together as a `usize` and compare those, and then compare the remaining parts of the string if that first word of memory is equal. We don't use the `Eq` optimization in Spellbook since Umbra strings are only used for the lookup table representation (we use `PartialEq<str> for UmbraString` instead), but it's interesting from an academic perspective.

```rust
impl PartialEq<Self> for UmbraString {
    fn eq(&self, other: &Self) -> bool {
        let self_len_and_prefix = ptr::from_ref(self).cast::<usize>();
        let other_len_and_prefix = ptr::from_ref(other).cast::<usize>();
        if unsafe { *self_len_and_prefix != *other_len_and_prefix } {
            return false;
        }

        // The lengths and prefixes are equal. Now compare the rest.
        if self.len <= INLINE_LEN {
            // We can use the same trick as above: compare the suffixes as one big chunk.
            let self_ptr = ptr::from_ref(self);
            let self_suffix = unsafe { ptr::addr_of!((*self_ptr).trailing.suffix) }.cast::<usize>();
            let other_ptr = ptr::from_ref(other);
            let other_suffix = unsafe { ptr::addr_of!((*other_ptr).trailing.suffix) }.cast::<usize>();

            unsafe { *self_suffix == *other_suffix }
        } else {
            let suffix_len = self.len() - PREFIX_LEN;
            let self_rest = unsafe {
                slice::from_raw_parts(
                    self.trailing.ptr.as_ptr().add(PREFIX_LEN),
                    suffix_len
                )
            };
            let other_rest = unsafe {
                slice::from_raw_parts(
                    other.trailing.ptr.as_ptr().add(PREFIX_LEN),
                    suffix_len
                )
            };

            self_rest == other_rest
        }
    }
}

impl Eq for UmbraString {}
```

We start by comparing the length and prefix parts together with one `usize` comparison. If that is equal then we need to check the rest. For the short variant we can use another `usize` comparison to check the rest. For the long variant we can reconstruct the byte slices for the remaining bytes and compare those.

We can actually make this a little better. We know in that `else` block that the lengths of `self` and `other` are equal but comparing the byte slices (`PartialEq<Self> for &[T]`) will repeat that check. We can skip that check and do the comparison directly. Since `u8`s are byte-wise equal to each other, we can use `memcmp` like the standard library does.

```rust
impl PartialEq<Self> for UmbraString {
    fn eq(&self, other: &Self) -> bool {
        // ... unchanged ...
        if self.len <= INLINE_LEN {
            // ... unchanged ...
        } else {
            let suffix_n_bytes = self.len() - PREFIX_LEN;
            unsafe {
                memcmp(
                    self.trailing.ptr.as_ptr().add(PREFIX_LEN),
                    other.trailing.ptr.as_ptr().add(PREFIX_LEN),
                    suffix_n_bytes,
                ) == 0
            }
        }
    }
}

// Snipped from `library/core/src/slice/cmp.rs`:
extern "C" {
    /// Calls implementation provided memcmp.
    ///
    /// Interprets the data as u8.
    ///
    /// Returns 0 for equal, < 0 for less than and > 0 for greater
    /// than.
    fn memcmp(s1: *const u8, s2: *const u8, n: usize) -> core::ffi::c_int;
}
```

# Benchmarking and memory analysis

Speed benchmarks are unfortunately not very interesting. Spellbook doesn't take advantage of the `Eq` comparison so we only end up paying for the conversion in `UmbraString::as_slice`. This is nearly imperceptibly slower than `Box<str>::as_bytes`. Using `cargo bench` and simple benchmarks like so:

```rust
// NOTE: this needs nightly.
#![feature(test)]
extern crate test;
use test::{black_box, Bencher};
use spellbook::umbra_slice::UmbraString;

#[bench]
fn umbra_str_as_bytes(b: &mut Bencher) {
    let s: UmbraString = "a".repeat(50).into();
    b.iter(|| black_box(&s).as_bytes());
}
#[bench]
fn boxed_str_as_bytes(b: &mut Bencher) {
    let s: Box<str> = "a".repeat(50).into();
    b.iter(|| black_box(&s).as_bytes());
}
```

`umbra_str_as_bytes` measures at around 0.69 ns/iter on my machine while `boxed_str_as_bytes` measures around 0.46 ns/iter. We would need to be converting to bytes very very often to notice the difference, and Spellbook doesn't ultimately convert that often. The benchmarks for Spellbook's `check` function don't change perceptibly.

Where we see the difference is in memory usage and heap interaction. Measuring heap allocations is not as straightforward in Rust as you might imagine if you're coming from garbage collected languages: garbage collectors need to track the heap to know when to clean up garbage so there's typically an interface to query heap information. Not so with Rust. [Measuring Memory Usage in Rust](https://rust-analyzer.github.io/blog/2020/12/04/measuring-memory-usage-in-rust.html) from the `rust-analyzer` blog points out a few options. Of them I'm partial to `valgrind`'s [DHAT](https://valgrind.org/docs/manual/dh-manual.html) tool since it's straightforward to use.

We'll run a small example program that creates the `en_US` dictionary and checks a single word:

```sh
cargo run --release --example check hello
valgrind --tool=dhat ./target/release/examples/check hello
```

Before (`Box<str>` stems), DHAT reports:

```
Total:     3,086,190 bytes in 130,988 blocks
At t-gmax: 2,717,005 bytes in 90,410 blocks
At t-end:  0 bytes in 0 blocks
Reads:     3,923,475 bytes
Writes:    2,610,900 bytes
```

After (`UmbraString` stems):

```
Total:     2,714,546 bytes in 82,475 blocks
At t-gmax: 2,343,567 bytes in 41,487 blocks
At t-end:  0 bytes in 0 blocks
Reads:     2,332,587 bytes
Writes:    2,239,256 bytes
```

We've saved around 300kb of total runtime memory (12%) with the change, plus we're using fewer blocks of memory and reading from and writing to the heap less. Success!

We can go further though if we apply this "German string" optimization to another oft-instantiated type in the lookup table: the `FlagSet`.

# Bonus points: the FlagSet can also be German!

Remember way back at the beginning of the post when were discussing the lookup table and how it's like a `HashMap<String, HashSet<Flag>>`? The `HashSet<Flag>` part is defined in the Spellbook source as a `FlagSet` newtype wrapper. It doesn't wrap a `HashSet<Flag>` though - hash sets can be wasteful in terms of memory usage. Before the Umbra string optimization they were represented as `Box<[Flag]>`. For short slices, `slice::contains` or `slice::binary_search` are very fast at determining set membership.

Like stems, flagsets are usually short. If we measure a histogram of the number of flags used per stem in all dictionaries in `LibreOffice/dictionaries`, we see the distribution skew **very** short:

| Number of flags | Percentile (rounded) |
|-----------------|----------------------|
| 0               | 32                   |
| 1               | 69                   |
| 2               | 80                   |
| 3               | 86                   |
| 4               | 90                   |
| ...             | ...                  |
| 7               | 96                   |
| ...             | ...                  |

One crazy dictionary used 271 flags on a single stem.

So if we can store some number of flags inline like we did with bytes an Umbra string, we can avoid allocations in the vast majority of cases.

Rather than an "Umbra string" we'll be constructing a more generic "Umbra slice" type. In fact we can imagine that the `UmbraString` is just a special case of an `UmbraSlice` around bytes:

```rust
// These bytes are valid UTF-8.
struct UmbraString(UmbraSlice<u8>);
```

The new type comes with new challenges though. For... _reasons_... `Flag` is defined as:

```rust
type Flag = core::num::NonZeroU16;
```

So rather than dealing with bytes we need to deal with 16-bit integers. Ok, that changes the arithmetic a little:

```rust
// We can fit 3 u16s in the prefix.
const fn prefix_len<T>() -> usize {
    // Remove 16 bits for the `len`.
    (size_of::<usize>() - size_of::<u16>()) / size_of::<T>()
}
// And 4 in the suffix.
const fn suffix_len<T>() -> usize {
    size_of::<usize>() / size_of::<T>()
}
```

We can fit up to 7 flags inline. That's really awesome: it'll cover up to 96% of real-world flagsets and should save us many many really tiny allocations.

## Pitfalls and `MaybeUninit<T>`

We're talking in terms of `u16` above but our type is actually a `NonZeroU16`. They have the same size and layout but `NonZeroU16` can't be `0u16`. The challenge is the `NonZero` nature: the zeroed bit pattern is not a valid representation, and `Default for NonZeroU16` is not a thing. Places where we wrote `[0u8; N]` above have to be rewritten anyways since we're changing the type, but we can't just say:

```rust
// 💣 UNDEFINED BEHAVIOR!!
let mut prefix: [T; PREFIX_LEN] = unsafe { core::mem::zeroed() };
let mut suffix: [T; SUFFIX_LEN] = unsafe { core::mem::zeroed() };
```

You can't say that a value is a `NonZeroU16` and at the same time represent it with zeroes, even if you never formally access those elements of the array. The proper way to encode what we're trying to do is to use [MaybeUninit](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html).

```rust
use core::mem::MaybeUninit;
use crate::Flag;

// Unfortunately we cannot call `prefix_len`/`suffix_len` within
// the definition of `UmbraSlice` so we need to use const generics.
// The result is that this type is not pretty :/
pub type FlagSlice = UmbraSlice<
    Flag,
    { prefix_len::<Flag>() },
    { suffix_len::<Flag>() },
>;

#[repr(C)]
pub struct UmbraSlice<T: Copy, const PREFIX_LEN: usize, const SUFFIX_LEN: usize> {
    len: u16,
    prefix: [MaybeUninit<T>; PREFIX_LEN],
    trailing: Trailing<T, SUFFIX_LEN>,
}

#[repr(C)]
union Trailing<T: Copy>, const SUFFIX_LEN: usize> {
    suffix: [MaybeUninit<T>; SUFFIX_LEN],
    ptr: ManuallyDrop<NonNull<T>>,
}

impl<T: Copy, const PREFIX_LEN: usize, const SUFFIX_LEN: usize>
    UmbraSlice<T, PREFIX_LEN, SUFFIX_LEN>
{
    const INLINE_LEN: u16 = (PREFIX_LEN + SUFFIX_LEN) as u16;
}
```

This makes the type slightly harder to work with: when accessing the `prefix` and `suffix` arrays we need to be sure to `ptr::cast()` from a pointer of `MaybeUninit<T>` to a pointer of `T`. When initializing the slice in our `From` implementation we need to transmute the source slice from `&[T]` to `&[MaybeUninit<T>]` before we can copy the data:

```rust
fn copy_to_slice<T: Copy>(dst: &mut [MaybeUninit<T>], src: &[T]) {
    // SAFETY: &[T] and &[MaybeUninit<T>] have the same layout.
    let uninit_src: &[MaybeUninit<T>] = unsafe { core::mem::transmute(src) };
    dst.copy_from_slice(uninit_src);
}
```

## Zeroed bit patterns

We also need to be **very** careful to initialize `prefix` and `suffix` with `MaybeUninit<T>::zeroed()` rather than `MaybeUninit<T>::uninit()`. Why? Remember that our `PartialEq<Self>` implementation compares the prefix array and maybe also the suffix array for the short variant. Those arrays might contain uninitialized data if the length of the slice is shorter than the `INLINE_LEN` or `PREFIX_LEN`. `MaybeUninit<T>::zeroed()` works around this because comparing zeroed bits is defined behavior. The important distinction is that we are not treating the zeroed memory as `NonZeroU16`. That is undefined behavior. If we treat it as a `usize` though, the zeroed bit pattern is valid and the behavior is defined. It's also accurate as long as `T` is `Copy`.

```rust
// Note that `T` is not bound by `Eq`.
// We only ever compare bits, not `T`s.

impl<T: Copy, const PREFIX_LEN: usize, const SUFFIX_LEN: usize> PartialEq<Self>
    for UmbraSlice<T, PREFIX_LEN, SUFFIX_LEN>
{
    fn eq(&self, other: &Self) -> bool {
        // SAFETY: the `prefix` field is created with `MaybeUninit::zeroed` memory, so even
        // if the slice has fewer than `PREFIX_LEN` elements, comparing the uninitialized
        // memory is defined behavior, and it is accurate since `T` is `Copy`.
        let self_len_and_prefix = ptr::from_ref(self).cast::<usize>();
        let other_len_and_prefix = ptr::from_ref(other).cast::<usize>();
        if unsafe { *self_len_and_prefix != *other_len_and_prefix } {
            return false;
        }

        // ... compare suffixes ...
    }
}
```

## Null pointer optimization and strange behavior

What exactly can go wrong if you don't use `MaybeUninit<T>`? The compiler can see that `NonZeroU16` cannot ever be a zeroed bit pattern and it can design the layouts for other types using `FlagSlice` around that. If we designed our type like this:

```rust
#[repr(C)]
struct FlagSlice {
    len: u16,
    prefix: [Flag; PREFIX_LEN],
    trailing: Trailing,
}

#[repr(C)]
union Trailing {
    suffix: [Flag; SUFFIX_LEN],
    ptr: ManuallyDrop<NonNull<Flag>>,
}
```

Then `FlagSlice` is eligible for the [null pointer memory layout optimization](https://doc.rust-lang.org/std/option/index.html#representation). The compiler can tell that the zero bit pattern is not a valid representation for the struct and so it can try to fit other information in that representation, like whether an `Option<T>` is `Some` or `None`. It's a really handy optimization that makes `size_of::<Option<T>>() == size_of::<T>()` - you don't pay for the option. But how would you represent the empty flag slice?

```rust
// 💣 UNDEFINED BEHAVIOR!!
impl Default for FlagSlice {
    fn default() -> Self {
        Self {
            len: 0,
            prefix: unsafe { core::mem::zeroed() },
            trailing: Trailing {
                suffix: unsafe { core::mem::zeroed() },
            }
        }
    }
}
```

The length is zero, the prefix is zeroes, the suffix is zeroes. The whole struct is zeroes! With this representation, `Option::<FlagSlice>::None` is exactly the same as `FlagSlice::default()`, causing your code to behave _weirdly_. Suddenly `Some(FlagSlice::default()).is_some()` is `false`! 🥴

While this pitfall seems scary and hard to debug, [Miri](https://github.com/rust-lang/miri) has got your back. Write types without the `MaybeUninit<T>` wrapper and `cargo miri test` will helpfully point out that you're opening yourself up to undefined behavior.

## FlagSlice Memory Savings

Rerunning the same example from above, DHAT reports:

```
Total:     2,584,850 bytes in 44,741 blocks
At t-gmax: 2,190,833 bytes in 947 blocks
At t-end:  0 bytes in 0 blocks
Reads:     1,733,361 bytes
Writes:    2,109,560 bytes
```

So to compare:

| Stem + FlagSet                | Total                             | At t-gmax                        | Reads (B) | Writes (B) |
|-------------------------------|-----------------------------------|----------------------------------|-----------|------------|
| `Box<str>` + `Box<[Flag]>`    | 3,086,190 bytes in 130,988 blocks | 2,717,005 bytes in 90,410 blocks | 3,923,475 | 2,610,900  |
| `UmbraString` + `Box<[Flag]>` | 2,714,546 bytes in 82,475 blocks  | 2,343,567 bytes in 41,487 blocks | 2,332,587 | 2,239,256  |
| `UmbraString` + `FlagSlice`   | 2,584,850 bytes in 44,741 blocks  | 2,190,833 bytes in 947 blocks    | 1,733,361 | 2,109,560  |

These are some respectable savings! We've cut out about a half of a megabyte of total memory, used far fewer allocations (blocks) and write to the heap a fair amount less. Plus we read from the heap less than half as much as we did before the changes.

Not every dictionary will see the same savings, though: some dictionaries use more flags and have longer stems. But as discussed above, every time we use a short variant of an Umbra slice we save memory over a `Box<str>` or `Box<[Flag]>`.

# Wrapping up & Kudos

We've designed and implemented a German string inspired `UmbraSlice<T>` type that can carry a small number of `T`s inline - a small _slice_ optimization - and used it to save a respectable amount of total memory for the `Dictionary` type, and also cut way down on heap interaction. We've also stumbled upon lots of interesting detours into Rust topics: fat pointers, runtime memory measurement, `MaybeUninit<T>` and the null-pointer optimization. The full code for `UmbraSlice<T>` lives in Spellbook's repository in [`src/umbra_slice.rs`](https://github.com/helix-editor/spellbook/blob/master/src/umbra_slice.rs).

As mentioned above, CedarDB has an excellent [intro post for German strings](https://cedardb.com/blog/german_strings/) and also a nice [deeper dive](https://cedardb.com/blog/strings_deep_dive/). The former has a snide remark about an optimization which is supposedly impossible in Rust, provoking interesting response posts by those who had been successfully nerd-sniped. One of these - [An Optimization That's Impossible in Rust!](https://tunglevo.com/note/an-optimization-thats-impossible-in-rust/) - I found very informative on the Rust aspects of implementing German strings, and may be interesting if your use-case benefits from `Clone for UmbraString` being cheap like `Clone for Arc`. (Not so for Spellbook.) Thank you to these authors!
