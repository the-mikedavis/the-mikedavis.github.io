+++
title = "Sparse arrays in Rust and creating custom DSTs"
date = 2025-07-04
taxonomies.tags = ["Rust", "data structures"]

[extra]
repo_view = true
+++

[Dynamically sized types](https://doc.rust-lang.org/reference/dynamically-sized-types.html) (DSTs) are one of a few _[exotically sized](https://doc.rust-lang.org/nomicon/exotic-sizes.html)_ Rust types. Rust and its standard library have good tools for creating regular sized types but creating dynamically sized types can be a pain.

We'll jump through DSTs quickly and look at an application - _sparse arrays_ - to see how tricky DSTs are today.

While creating a DST I was surprised at the lack of resources on DST creation and use. This post is meant as a reference for anyone retracing my steps, or as an informative look into a trickier part of data structure implementation in Rust.

This post is meant for those interested in data structure design in Rust and, to get the most out of this post, you should already be familiar with the concept of a DST. Otherwise this post may be a bit in-the-weeds - but hopefully still valuable.

### Dynamically sized types

DSTs are special because the compiler doesn't know how large they are at runtime. Take a regular _sized_ type:

```rust
struct Color {
    red: u8,
    green: u8,
    blue: u8,
}
```

The compiler knows the size of this type. It's the sum of it's fields, 3 bytes.

What about `[u8]` though, how many bytes are in that? If it's not a constant-sized array like `[u8; 3]` then you don't know at compile-time.

### DSTs in the wild

The two common types of DSTs you may have seen are `str` and `[T]`. In fact `str` is just `[u8]` that's guaranteed to be UTF-8 so let's just say `[T]`.

Unlike `[T; 5]`, `[T]` has an unknown number of `T`s. In practice you've probably seen this type borrowed: `&[T]`. There are also owned versions: `Box<[T]>`, `Rc<[T]>`, `Arc<[T]>` and similar.

The borrowed version is intuitive: you have a reference to some consecutive elements in an array allocation. A `&[T]` is a slice of some number of elements of a `Vec<T>` (or similar array allocation). And a `&str` is a slice of some valid UTF-8 bytes of a `String` (`String` is just a `Vec<u8>` that is valid UTF-8).

The owned ones are not much different. They're really just owned: the data is on the heap and the type represents a pointer to it which has ownership of the allocated data. The trick, though, is that these types are _wider_ than usual:

```rust
// NOTE: doesn't matter what type `T` is.
assert_eq!(size_of::<Box<[T]>>(), 2 * size_of::<usize>());
// Same with the borrowed kind
assert_eq!(size_of::<&[T]>(), 2 * size_of::<usize>());
```

If it were just a pointer it would be the same size as a `usize`: a machine word, the size of a pointer by definition. Instead `Box<[T]>` and friends take two words to represent. One word is the pointer to the data and the other is the length of the slice.

### An application: sparse arrays

I'll be talking about sparse arrays as they are used in the use-case I'm interested in: [Hash Array Mapped Trie](https://en.wikipedia.org/wiki/Hash_array_mapped_trie)s. In this context, a sparse array is an array - a contiguous sequence of values - where you expect a "sparse" number (i.e. few) entries to actually be used. The rest of the slots are left uninitialized or as defaults.

If you're familiar with `MaybeUninit<T>`, a sparse array is conceptually like `[MaybeUninit<T>; N]`. The upside of sparse arrays is that you don't pay the cost of representing uninitialized slots.

Say you have a `[T; 64]` - 64 `T`s, stored contiguously in memory. If only some of these are initialized or meaningful then you waste a lot of space representing the entire array. Only utilizing some small `N` indices of this array leaves `64 - N` slots wasted. When you have very many of these arrays or when `size_of::<T>()` is large, wasted `T` allocations can add up.

Sparse arrays cut down on this wasted space by only allocating enough space to store occupied slots. Then a header is used to determine which indexes into the array are occupied by which slots. For Hash Array Mapped Tries, the layout looks like this:

```rust
#[derive(Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
struct Bitmap(u64);

#[repr(C)]
struct SparseArray<T> {
    bitmap: Bitmap,
    entries: [T],
}
```

At the beginning of this type, a 64-bit bitmap describes which entries are set in the dynamically sized array, and which indexes into the `SparseArray` correspond to which index in `entries`. With this layout, setting index `5` to `100` and `10` to `200` means that the 5th and 10th bit of the bitmap are set and the entries array is allocated with `100` and then `200`.

You can imagine an empty sparse array as just the bitmap:

```
+------+
+ 0000 +
+------+
```

And then a sparse array with elements at index 0 and 3 could be:

```
+------+------+------+
+ 1001 +  T   +  T   +
+------+------+------+
```

This use-case fits a dynamically sized type well: the true size of the type depends on its value at runtime. The number of bits set in the bitmap is the number of entries in `entries`.

This takes a fixed size of overhead for every sparse array, but you don't pay for allocating uninitialized or meaningless entries.

This post skips over the implications of inserting or deleting an element (resizing the sparse array) and other interesting features. I will cover Hash Array Mapped Tries in part two of this post. Instead here we'll talk about how implementing this type - and using it - is a pain, and how we can make our lives easier by just giving up and dealing with pointers.

### Fighting for a DST - a fool's errand

My first implementation of this type looked just like the above `SparseArray<T>`:

```rust
#[repr(C)]
struct SparseArray<T> {
    bitmap: Bitmap,
    entries: [T],
}
```

Note that `entries: [T]` is what makes it a DST and will bring us much pain. (As a peculiarity of custom DSTs, it must be the last field which is a DST. A DST anywhere else is an error.) The compiler can see from `[T]` that we're dealing with an "unsized" type (`?Sized`) and will let us know in future error messages.

We'll start with making an empty `SparseArray`.

```rust
impl Bitmap {
    // No bits set means no entries.
    const EMPTY: Self = Self(0);
}

impl<T> SparseArray<T> {
    pub fn empty() -> Self {
        Self { bitmap: Bitmap::EMPTY, entries: [] }
    }
}
```

But already we run into trouble:

```
error[E0277]: the size for values of type `[T]` cannot be known at compilation time
  --> src/lib.rs:15:23
   |
15 |     pub fn empty() -> Self {
   |                       ^^^^ doesn't have a size known at compile-time
   |
   = help: within `SparseArray<T>`, the trait `Sized` is not implemented for `[T]`
```

The gist is that we can't return `SparseArray<T>` from any function. This is the same for familiar DSTs too: you can't have a variable bound to some `[T]`. Instead you always work in terms of `&[T]` or `Box<[T]>` (or similar owned wrapper).

```rust
// 💣  &[T] is ok but not [T]
fn count<T>(arr: [T]) -> usize {
    arr.iter().count()
}
```

```
error[E0277]: the size for values of type `[T]` cannot be known at compilation time
 --> src/lib.rs:1:16
  |
1 | fn count<T>(arr: [T]) -> usize {
  |                ^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `[T]`
help: function arguments must have a statically known size, borrowed slices always have a known size
  |
1 | fn count<T>(arr: &[T]) -> usize {
  |   
```

A consequence of being unable to return `Self` is that we can't implement useful traits like `Clone` and `Default`. So already we are drifting away from the the nice tools available for sized types.

The solution here? Return an owned type wrapping `SparseArray`. Like mentioned above we can use a `Box<Self>`, `Rc<Self>`, `Arc<Self>` or anything similar. I'll use `Box<Self>` here for simplicity, though eventually I want an `Arc` to make this data structure persistent and thread-safe.

```rust
use std::{
    alloc::{self, Layout},
    ptr::{self, NonNull},
};

impl Bitmap {
    const fn len(&self) -> usize {
        self.0.count_ones() as usize
    }
}

impl<T> SparseArray<T> {
    pub fn empty() -> Box<SparseArray<T>> {
        let bitmap = Bitmap::EMPTY;
        let layout = Self::layout(bitmap.len());
        let nullable = unsafe { alloc::alloc(layout) };
        let ptr = match NonNull::new(nullable) {
            Some(ptr) => ptr.as_ptr(),
            None => alloc::handle_alloc_error(layout),
        };
        // This is necessary in order to be able to cast!!
        let ptr = ptr::slice_from_raw_parts_mut(ptr, len) as *mut Self;

        unsafe { (&raw mut ((*ptr).bitmap)).write(bitmap) }
        // TODO: we'd then write entries, if there were any.
        unsafe { Box::from_raw(ptr) }
    }

    fn layout(len: usize) -> Layout {
        Layout::new::<Bitmap>()
            .extend(Layout::array::<T>(len).unwrap())
            .unwrap()
            .0
            .pad_to_align()
    }
}
```

Reaching into pointer and allocation stuff from `std::alloc` and `std::ptr` may look a little ham-fisted but it's not necessarily unusual for data structure implementations. The part of this that was unfamiliar to me was [`std::ptr::slice_from_raw_parts_mut`](https://doc.rust-lang.org/std/ptr/fn.slice_from_raw_parts.html). This is one of the standard library's few utilities for working with dynamically sized types. The important part about this function is that it forms a possibly wide pointer based on the metadata (second argument). A _wide pointer_ describes the pointer itself along with some metadata like the number of elements in a `[T]`.

So with `empty` or `new`, `Box<SparseArray<T>>` actually ends up as `size_of::<usize>() * 2`. One word holds the pointer to the DST and another holds the length of `entries`. Unfortunately this is using extra space to encode data that we already know. The number of entries can be computed from the bitmap by counting the number of 1 bits, either through CPU-specific instructions, a series of shifts or SIMD.

Already I don't like adding unnecessary data just to track the DST. And as mentioned above, we're fighting common traits. If we try to reimplement `Clone`:

```rust
impl<T: Clone> SparseArray<T> {
    fn clone(this: &Self) -> Box<Self> {
        // TODO: `new` is like `empty` but writes the `entries` too
        // based on the iterator
        Self::new(this.bitmap, this.entries.iter().cloned())
    }
}
```

This doesn't look very much like a regular `impl Clone for T`. And to call it we need to use `SparseArray::clone(&my_sparse_array)` :/. We've left familiar Rust ergonomics!

### The workaround: pointers

We're straying from the niceties of sized types and that leaves us doing un-ergonomic stuff. Can we make things simpler for ourselves, and maybe avoid that extra wide-pointer overhead, please?

#### The type

`entries: [T]` is a problem because it's a DST. Let's replace it with a sized type... `[T; 0]`.

```rust
impl Bitmap {
    const MAX_ENTRIES: usize = u64::BITS
}

#[repr(C)]
struct SparseArrayInner<T> {
    bitmap: Bitmap,
    // HEY! This location is a sequence of `0..Bitmap::MAX_ENTRIES`
    // `T`s. `read`/`write` it accordingly!!
    entries: [T; 0],
}

pub struct SparseArray<T>(NonNull<SparseArrayInner<T>>);
```

Now `SparseArray<T>` is a pointer to some allocated type and we've chosen to hide the actual size of `SparseArrayInner` from the compiler. The tradeoff is that we are now responsible for allocating the data and accessing it appropriately. And now `SparseArray<T>` is a sized type: `NonNull<T>` is a pointer and we know the size of a pointer statically.

#### Accessing the bitmap and entries

If the `SparseArray<T>` type is a pointer to the inner type, we can read that pointer to get the `Bitmap` and `&[T]`.

```rust
impl<T> SparseArray<T> {
    pub fn bitmap(&self) -> Bitmap {
        unsafe { self.0.read() }.bitmap
    }

    pub fn entries(&self) -> &[T] {
        let len = self.bitmap().len();
        let ptr = unsafe { &raw const (*self.0.as_ptr()).entries }.cast::<T>();
        unsafe { std::slice::from_raw_parts(ptr, len) }
    }
}
```

In `SparseArray::entries` we cast the pointer to the entries, `*const [T; 0]`, to be a `*const T`. This lets us create a slice from it. The safety docs for `slice::from_raw_parts` say that the pointer must be valid for a read of `len` `T`s. As long as we allocate and initialize those entries then we're good to go.

#### Allocating the sparse array

Let's rewrite `SparseArray::new` from above to allocate the memory, but then cast it as a `SparseArrayInner<T>` rather than deal with `ptr::slice_from_raw_parts_mut`.

```rust
impl<T> SparseArray<T> {
    pub fn new<I: IntoIterator<Item = T>>(bitmap: Bitmap, entries: I) -> Self {
        let len = bitmap.len();
        let layout = Self::layout(len);
        let nullable = unsafe { alloc::alloc(layout) };
        let non_null = match NonNull::new(nullable) {
            Some(ptr) => ptr.cast::<SparseArrayInner<T>>(),
            None => alloc::handle_alloc_error(layout),
        };
        let ptr = non_null.as_ptr();
        unsafe { (&raw mut ((*ptr).bitmap)).write(bitmap) }
        let entries_ptr = unsafe { &raw mut ((*ptr).entries) as *mut T };
        for (i, entry) in entries.iter().enumerate() {
            unsafe { entries_ptr.add(i).write(entry) };
        }
        Self(non_null)
    }

    fn layout(len: usize) -> Layout {
        Layout::new::<Bitmap>()
            .extend(Layout::array::<T>(len).unwrap())
            .unwrap()
            .0
            .pad_to_align()
    }
}
```

We allocate for enough `T`s according to the `Bitmap::len`. Then, like above, we cast the pointer to the entries, `*mut [T; 0]`, to `*mut T` and then perform the writing of contiguous entries ourselves.

#### `Default`

Now that `SparseArray<T>` is a sized type, we can create them ergonomically with familiar traits like `Default`. This replaces our prior `SparseArray::empty` function.

```rust
impl<T> Default for SparseArray<T> {
    fn default() -> Self {
        Self::new(Bitmap::EMPTY, [])
    }
}

```

That's more like it! Very straightforward.

#### `Clone`

While we're at it, implementing `Clone` is straightforward too: copy the bitmap and clone all of the entries.

```rust
impl<T: Clone> Clone for SparseArray<T> {
    fn clone(&self) -> Self {
        Self::new(self.bitmap(), self.entries().iter().cloned())
    }
}
```

As mentioned above I want to use this type in a Hash Array Mapped Trie and I want to put the pointer behind an `Arc`. `Clone` is a really useful standard trait to implement for this use-case since it will allow us to use the awesome standard library function [`Arc::make_mut`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.make_mut) which makes concurrent persistent data structures easy to implement.

#### `Drop`

The really important trait to implement is `Drop`. Since we've taken over the allocation of this type, the compiler will no longer help us drop it correctly.

```rust
impl<T> Drop for SparseArray<T> {
    fn drop(&mut self) {
        if std::mem::needs_drop::<T>() {
            // NOTE: `entries_mut` is mostly the same as `entries`
            // but uses `from_raw_parts_mut` instead.
            for entry in self.entries_mut() {
                let ptr = entry as *mut _;
                unsafe { ptr::drop_in_place(ptr) };
            }
        }
        let layout = Self::layout(self.bitmap().len());
        unsafe { alloc::dealloc(self.ptr.as_ptr().cast(), layout) }
    }
}
```

When dropping a `SparseArray<T>` we first drop the entries, if they need to be dropped, and then deallocate the chunk of memory.

### Wrapping up

With the new definition of `SparseArray<T>` we have a sized type which can implement common traits like `Default` and `Clone`. While the actual allocated chunk of memory has a variable size, the size of a `SparseArray<T>` is constant and _thin_ - only the size of a pointer. Now we don't waste an extra `usize` of memory on tracking the `entries` length twice.

```rust
assert_eq!(size_of::<SparseArray<u8>>(), size_of::<usize>());
// The size of the inner type doesn't matter!
assert_eq!(size_of::<SparseArray<u128>>(), size_of::<usize>());
```

While custom sized types are well supported by the standard library, dynamically sized types (DSTs) can be annoying to create and work with. An escape hatch for this is to put the type behind a pointer yourself. This places more responsibility on you, but it gives you full control over the type's representation.
