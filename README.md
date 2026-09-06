# ps-thread-local-no-std

`ThreadLocal<T>`, per object, without linking `std`.

A fork of [`os-thread-local`](https://crates.io/crates/os-thread-local) 0.1.3 by
Mike Hommey, dual licensed Apache-2.0/MIT, unchanged except for five lines. The
upstream crate was already built entirely on OS primitives; it just said `std`
in three places where it meant `core` and `alloc`.

## Why this exists

The same eighty lines of `pthread_key_create` plumbing had been hand-rolled
twice in this house, in `parking_lot_lite_hack_core` and in `ps-reclaim`, with
`WorkTablesIndex` about to make it three. Each copy is a separate place to get
key racing, destructor registration and teardown-time failure wrong. This is one
copy, with somebody else's 1.16 million downloads behind it.

## What changed from upstream

```rust
#![cfg_attr(not(test), no_std)]     // added
extern crate alloc;                 // added
use core::error::Error;             // was std::error::Error
use alloc::boxed::Box;              // was std::boxed::Box
```

plus the crate name in five doctests. Nothing else. The `pthread` and
`FlsAlloc` code, the destructor handling, the tests and the documentation are
upstream's, and a diff against `os-thread-local` 0.1.3 should stay this short so
that upstream fixes remain trivial to carry across.

`not(test)` rather than a bare `no_std` because the test module wants threads and
channels. Gating it off would have left the `no_std` build as the one nobody
tests; this way `cargo test` links `std` for the harness while the crate's own
code stays on `core` plus `alloc`.

`core::error::Error` is stable since 1.81, which is the crate's floor.

## Yes, a crate wrapping libc is `no_std`

`libc` is a bindings crate and is `#![no_std]` with `default-features = false`,
which is how it is depended on here. On musl specifically it is fine, with one
limit worth knowing: musl defines `PTHREAD_KEYS_MAX` as **128** where glibc
allows 1024, and this crate takes one key per `ThreadLocal` object rather than
one per program. `ThreadLocal::new` asserts on `pthread_key_create` failing, so
a program that creates thousands of these on musl panics rather than degrades.
For a handful of long-lived statics, which is what this exists for, 128 is not a
ceiling anyone reaches.

Capping ourselves at 128 to match musl was considered and rejected. It does not
make glibc safe, it makes glibc fail earlier to match the worst platform, and it
does not even buy consistency: musl's 128 is a process-wide budget shared with
libc and every other library, so an internal count reaching 127 says nothing
about whether a key is left. If per-object thread-locals ever need to scale, the
answer is the one `thread_local` and `seize` use, a single key holding a
per-thread table indexed by object id, which costs one indirection and makes the
platform limit irrelevant. Nothing in this house needs it: every consumer holds
its keys in `static`s, four of them in `ps-reclaim` and one in
`parking_lot_lite_hack_core`. `no_std` means "does not link the Rust
standard library", not "makes no calls into C". The `dep:libc` here resolves to
symbols the platform's libc already exports into every process on these targets.

Going below it is a real option and a different project. `pthread_key_create` is
not a syscall, it is userspace bookkeeping in libc, so there is no `syscall`
instruction that replaces this file. The thing underneath is the thread pointer
itself: `%fs` on x86-64, `TPIDR_EL0` on aarch64, a `PT_TLS` segment and `TPOFF64`
relocations, set up by whoever loaded the program. Amos Wenger's
[part 13](https://fasterthanli.me/series/making-our-own-executable-packer/part-13)
walks through doing exactly that, and it is the loader's job, not a library's.
On macOS it is not available at all: Apple's syscall numbers are not a stable
interface and `libSystem` is the only supported entry point.

## Alternatives considered

| crate | why not |
|---|---|
| `std::thread_local!` | the thing being replaced; it is a `std` macro |
| [`thread_local`](https://crates.io/crates/thread_local) | not `no_std`, and keyed on `std::thread::current().id()` |
| [`lazy_thread_local`](https://crates.io/crates/lazy_thread_local) | genuinely `no_std` already, and the closest call. 4,882 total downloads, last release March 2024, MIT only where the house is dual, and it carries lazy-init machinery we do not need |
| `#[thread_local]` | still unstable, and it does not run destructors even on nightly |
| hand-rolling it again | what this crate exists to stop |

## What it costs

There is a hit. It is about half a nanosecond per access on this machine, and
you should know why before deciding whether that matters.

Twenty million accesses per arm, three rounds, aarch64 `apple-darwin`, every arm
behind `#[inline(never)]`, with arm A repeated as the null:

```text
      A const   B lazy   C static   D local   N const again
 r1     1.36     1.26     1.74      2.00        1.23
 r2     1.24     1.27     1.76      1.98        1.24
 r3     1.25     1.25     1.72      1.99        1.24
```

A is `thread_local!` with `const` init, B the same with a destructor and a lazy
flag, C this crate behind a `OnceLock` which is how a `static` would really use
it, D this crate through a reference passed in. **Roughly 1.25 ns against 1.74,
so about 0.5 ns or 40%.** Both are a call on Mach-O; `pthread_getspecific` does
more inside it than the TLV thunk, and `try_with` adds a `NonNull` check and a
guard comparison on top.

**Those numbers are the floor, not the answer.** The `#[inline(never)]` forbids
an optimisation that `thread_local!` legitimately gets and this crate can never
get, so it measures the two mechanisms at their most similar. Without it the
same benchmark reports `thread_local!` at 0.25 ns and a 5x win, and that is
**not** an artifact to be waved away. Compile a loop that increments one
`thread_local!` and LLVM deletes the loop:

```asm
blr  x9              ; one TLV resolve
ldr  x10, [x0]
add  x0, x10, x8     ; + n, closed form
str  x0, [x9]
```

Compile the same loop through this crate and every iteration survives, because
`pthread_getspecific` is an opaque extern call the optimiser has to assume can
do anything:

```asm
24: mov  x0, x20
28: bl   <pthread_getspecific>   ; every iteration
64: cmp  x21, #0x1               ; guard compare
6c: ldr  x8, [x21]
70: add  x0, x8, #0x1
78: subs x19, x19, #0x1
7c: b.ne 0x24                    ; the loop survives
```

So `thread_local!` is transparent to the optimiser and this crate is a barrier
to it. For scattered single accesses that costs the 40% above. For repeated
access in a loop it is unbounded. **The mitigation is free: put the loop inside
`with()` rather than `with()` inside the loop**, which is one lookup either way
and is how the hot paths in this house already use it.

**Not measured on Linux, and that is where the gap should be widest.** ELF
local-exec makes a `const`-initialised `thread_local!` a register read plus an
offset, with no call at all, while `pthread_getspecific` stays a call. Expect
worse than 40% there and measure before relying on this number.

For the caller this scales with the count of thread-locals on the path, not with
anything else: four of them on a pin path is four lookups, so about 2 ns added.
Folding several into one struct wins back more than the mechanism costs.

## Licence

Apache-2.0 OR MIT, upstream's, retained with upstream's copyright.
