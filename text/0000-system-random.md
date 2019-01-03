- Feature Name: system-random
- Start Date: 2018-12-21
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add `std::random::secure_random` and custom error type.

# Motivation
[motivation]: #motivation

This RFC was proposed in [Rand #648]. Provision of a secure source
of random numbers may be seen as a core Operating System service:

-   this is OS-specific, yet all major OSes and many emulation/sandbox
    environments have at least one randomness source
-   almost all `std` platforms have some form of randomness source provided by
    the OS, while most other platforms do not have standard solutions (although
    they may have device-specific ones)
-   providing random data is simple relative to all the things that can be done
    with it

Currently both the Rand's [`OsRng`] and `libstd`'s `hashmap_random_keys`
function (an internal detail used to implement [`RandomState`]) implement an
interface to random data provided by the current platform's interface. This is
a complex interface that must be duplicated since the `hashmap_random_keys`
function is not exposed from `std` (and not sufficently generic).

Additionally, this could be leveraged to provide uniform `no_std` support via
usage of [`lang_items`].

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `std::random` module provides the following function to access the current
platform's secure random interface:

```rust
/// Fill `buf` with random bytes from the platform's secure random number
/// source, or fail.
/// 
/// This interface is designed primarily with security in mind, and will fail if
/// there is no strong source of random data available. On failure no guarantees
/// are made regarding the contents of `buf`.
/// 
/// It is possible (though unlikely) that the implementation will block.
/// 
/// Note that the implementation may short-circuit on a zero-length buffer, so
/// to test initialisation you must request at least one byte.
pub fn secure_random(buf: &mut [u8]) -> Result<(), Error>
```

This module is also available as `core::random`, though `no_std` builds require
the user or a third-party crate to provide the backend.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## De-duplication and implementation

The current system random interface implementations have some differences:

| | std | Rand |
|-|-----|------|
|Unix| Use `getrandom` if available, otherwise use `/dev/urandom` | Use `getrandom` if available, otherwise test reading a byte from `/dev/random` in non-blocking mode and *if successful* use `/dev/urandom` |
|NetBSD| As Unix | Test reading a byte from `/dev/random` in non-blocking mode and *if successful* use `/dev/urandom` |
|DragonFly / Haiku / Emscripten| As Unix | Use `/dev/random` |
|Solaris| As Unix | Use `getrandom` if available, otherwise use `/dev/random` in non-blocking mode |
|SGX | Use RDRAND ten times | No support |
|CloudABI| Use `arc4random_buf` via `libc` | Use `random_get` via `cloudabi` crate |
|MacOS / iOS| Use `SecRandomCopyBytes` | Use `SecRandomCopyBytes` |
|FreeBSD| Use `kern.arandom` | Use `kern.arandom` |
|OpenBSD | Use `getentropy` | Use `getentropy` |
|Redox| Return constants | Use `rand:`  interface |
|Fuchsia| Use `zx_cprng_draw` | Use `fuchsia_zircon::cprng_draw` |
|Windows| Use `RtlGenRandom` | Use `RtlGenRandom` via `winapi` crate |
|WASM| Return constants | No support for WASM *alone*|
|WASM + `stdweb` | | Use browser or Node interface |
|WASM + `wasm-bindgen` | | Use browser or Node interface |

Additionally, Rand's `OsRng` splits large buffers into chunks at the maximum
size supported by each system interface.

We propose to unify these into a single `std::random::secure_random` function,
then implement [`RandomState`] and [`OsRng`] as wrappers around this.

## Blocking?

It is possible the random source may block (although we prefer `/dev/urandom`
over `/dev/random`). For example, this may happen if the RNG is not yet
initialised (which Rand's current implementation for Linux explicitly checks).

We have a choice here: in *some* cases where the source would block, we could
detect this and fail with a `WouldBlock` error code, although we cannot
guarantee the implementation will never block. Alternatively we could always
block. We could make this behaviour configurable.

Optionally, we could add a function like the following, which 
partially replaces the need for a non-blocking API:

```rust
/// Attempt to check, without blocking, whether the system random source is
/// ready for usage.
/// 
/// If this returns `Some(true)`, then calls to `secure_random` are likely to
/// succeed without blocking. If it returns `Some(false)`, they are likely to
/// block.
/// 
/// In case it is not possible to determine the status without blocking, this
/// function returns `None`.
fn is_ready() -> Option<bool>
```

## Error type

To enable this, we use a custom `Error` type, whose design is constrained to
enable `no_std` support. Since `core::random::Error` must be functional without
an allocator, we cannot use `std::io::Error` (which may use a `Box`). Since
`std::random::Error` must be the same type, we cannot copy the design of
`rand_core::Error`. We thus must pick a simple design, like the following:

```rust
pub enum ErrorKind {
    NotReady,   // optional status (see blocking section)
    Unavailable,
}

pub struct Error {
    pub msg: &'static str,  // cannot be a String or have 'a parameter as in &'a str
    pub kind: ErrorKind,
}
```

## Handling errors within `std`

The only (current) need for randomness within `libstd` is [`RandomState`].
Since a source of secure entropy is not critical except to avoid DoS attacks on
certain public services, we think it acceptable that `RandomState` use a
constant value should `secure_random` return an error. This allows use of
`HashMap` on all platforms.

Should this also use a constant if `secure_random` would block?

## `no_std` support

There are two challenges when `std` is not available:

1.  There may not be an allocator available. This is not an issue for
    `secure_random` itself, but does imply that the `std::io::Error` type cannot
    be used in its current form.
2.  The platform may not have a (reliable) secure randomness source.

We propose that `libcore` provide an implementation of `secure_random` which
always fails, but that `libstd` uses lang items to override this with a real
implementation, depending on the platform. Users of `no_std` must provide an
implementation themselves (likely via a third-party crate) or avoid dependence
on this function.

## WASM

WebAssembly targets web browsers, and these provide a JavaScript API to retrieve
secure random data (`crypto.getRandomValues`); alternatively, it may be possible
to use Node.js (`crypto.randomFillSync`). Since these are JavaScript functions,
some method of bridging Rust and JS is required; currently the Rand project
supports both `wasm-bindgen` and `stdweb`.

I am not sufficiently knowledgable to make a recommendation for what `std`
should do here; e.g. it could use `wasm-bindgen` or simply not implement
`secure_random` on WASM. **Help please**.

# Drawbacks
[drawbacks]: #drawbacks

It is worth pointing out that `std`'s current `hashmap_random_keys` is only
dependend on to provide randomisation of hash functions as protection against
DoS attacks on public services. Because of this, it is not considered
security-essential, especially on platforms which are unlikely to be used as a
web server. In light of this, the current implementations for Redox and WASM
simply use hard-coded constants. This would not be acceptable for the interface
proposed here, which may make supporting some platforms harder (e.g. while
Rand's [`OsRng`] does have support for WASM, it has been a frequent source
of issues) — although the API does allow for explicit failure.

# Unresolved questions

See the blocking section above and note under error handling.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could choose between multiple names for the interface, depending on what
we choose to emphasise: `secure_random`, `system_random`, `os_random`.
Personally I prefer `system_random` except for the possible association with
C#'s insecure generator.

We could use a struct instead of a plain function to objectify the
configuration. This allows the simple `SystemRandom::new().fill_buf(&mut buf)`
vs `SystemRandom::nonblocking().fill_buf(&mut buf)`. However, this appears
needlessly complex, although it does have some precedent in `std`:
`File::open(path)` vs `OpenOptions::new().read(true).open(path)`. This may also
require a different backend for `no_std` implementation via `lang_items`.

Optionally we could add a function like `random::is_ready() -> bool` to test
whether the system RNG is available and has been initialized. There is little
motivation though since the user can simply try reading a byte.

There is scope for supporting an *insecure* randomness source (via a separate
function, a parameter, or even an error code indicating that the buffer has been
filled with insecure random data).

We could alternatively provide an `Rng` trait and implement that, however this
is likely best left to an external crate. The [`RngCore`] trait arrived at in
Rand is complex in order to provide an optimal interface to multiple different
users.

Alternatively, we could not do this; the Rand crate is already widely used and
the new `rand_os` crate allows use of (almost) only a system-random wrapper.
This does not solve the duplicate implementation issue or solve the problem of
a user-provided alternative with `no_std` configurations (though the latter has
other solutions).

# Prior art
[prior-art]: #prior-art

This RFC is motivated by [Rand #648] and [Rand #643].

We can compare with other languages, though this has limited utility:

-   C advocates use of `srand(time(0))` for insecure seeding but has no
    equivalent to this proposal
-   C++ has `std::random_device` as a non-deterministic random number generator,
    where the specification allows implementation by a local PRNG, and
    implementations may or may not be secure (or may even be deterministic in
    buggy cases)
-   Python has a `os.urandom(size)` which wraps the OS randomness source (in
    blocking mode only) and is intended for cryptographic usage
-   Modern browsers support the JS API `crypto.getRandomValues(array);`
-   Java has more APIs than anyone could want
-   C# has `System.Random` which is *not* secure and `RNGCryptoServiceProvider`

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Details of the `Error` type

# Future possibilities
[future-possibilities]: #future-possibilities

(empty section)

[`OsRng`]: https://docs.rs/rand/0.6/rand/rngs/struct.OsRng.html
[`RandomState`]: https://doc.rust-lang.org/std/collections/hash_map/struct.RandomState.html
[Rand #648]: https://github.com/rust-random/rand/issues/648
[Rand #643]: https://github.com/rust-random/rand/pull/643
[`lang_items`]: https://doc.rust-lang.org/nightly/unstable-book/language-features/lang-items.html
[`RngCore`]: https://docs.rs/rand/0.6/rand/trait.RngCore.html
[`rand_core::Error`]: https://github.com/rust-random/rand/blob/master/rand_core/src/error.rs
