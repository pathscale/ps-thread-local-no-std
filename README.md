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
which is how it is depended on here. `no_std` means "does not link the Rust
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

The measurement that says the mechanism does not matter much is in
`ps-reclaim/src/tls.rs`: on Mach-O, `std`'s thread-local is itself a call through
a TLV descriptor, and `thread_local!` against `pthread_getspecific` is 1.17 to
1.71 ns against 1.36 to 1.41 ns, with the null moving 0.5 ns on a bad round.
Pick on maintainability, not on speed.

## Licence

Apache-2.0 OR MIT, upstream's, retained with upstream's copyright.
