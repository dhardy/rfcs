- Feature Name: system-random
- Start Date: 2018-12-21
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add `std::random::secure_random` and custom error type.

# Motivation
[motivation]: #motivation

Currently both the Rand's [`OsRng`] and `libstd`'s `hashmap_random_keys`
function (an internal detail used to implement [`RandomState`]) implement an
interface to random data provided by the current platform's interface. This is
a complex interface that must be duplicated.

Additionally, some users believe this interface should be exposed by the `std`
lib: see [Rand #648].

Finally, this could be leveraged to provide uniform `no_std` support via usage
of [`lang_items`].

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
/// Note that the implementation may short-circuit on a zero-length buffer, so
/// to test initialisation you must request at least one byte.
pub fn secure_random(buf: &mut [u8]) -> Result<(), Error>
```

To enable this, we use a custom `Error` type, whose design is constrained to
enable `no_std` support. This may be a copy of the [`rand_core::Error`] type.
Specifically, this type supports at least the following functionality (either
via methods or via fields as in the `rand_core` error type):

```rust
impl Error {
    /// Get a descriptive message of the error
    fn msg(&self) -> &'static str;

    /// Get a status code
    fn kind(&self) -> ErrorKind;
}

enum ErrorKind {
    /// The randomness source is not currently available (or does not have
    /// sufficient entropy to be secure), but is likely to be later
    NotReady,
    /// There is no functional secure randomness source
    Unavailable
}
```

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

## Error handling

The error handling type itself is sketched out above, but details may need
fleshing out.

The only (current) user of `secure_random` within `libstd` is [`RandomState`].
Since a source of secure entropy is not critical except to avoid DoS attacks on
certain public services, we think it acceptable that `RandomState` ignore
errors from `secure_random`. This allows use of `HashMap` on all platforms.

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

WebAssembly is tricky to support properly, since there is no support for random
numbers within the base standard itself. Nevertheless, it is often possible to
retrieve random numbers via a JavaScript API provided by the web browser or by
Node.js. This brings up a second problem: interaction with JavaScript requires
an external dependency (with `wasm-bindgen` and `stdweb` being the current
offerings).

I am not sufficiently knowledgable to make a recommendation here: **help please**.

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

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could choose between multiple names for the interface, depending on what
we choose to emphasise: `secure_random`, `system_random`.

We could alternatively provide an `Rng` trait and implement that, however this
is likely best left to an external crate. The [`RngCore`] trait arrived at in
Rand is complex in order to provide an optimal interface to multiple different
users.

Optionally we could add a function like `random::is_ready() -> bool` to test
whether the system RNG is available and has been initialized. There is little
motivation though since the user can simply try reading a byte.

There is scope for supporting an *insecure* randomness source (via a separate
function, a parameter, or even an error code indicating that the buffer has been
filled with insecure random data), however I am not aware of many insecure
sources (though the list might include a clock-based-RNG and RDRAND, depending
on point of view).

# Prior art
[prior-art]: #prior-art

This RFC is motivated by [Rand #648] and [Rand #643].

TODO: compare with other languages/libraries? **help**

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
