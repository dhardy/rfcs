- Feature Name: N/A
- Start Date: 2017-06-26
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

There has been a long-standing question around Rust's evolution: what exactly does
[stability without stagnation] mean in the medium-to-long term, and how will breaking
changes be introduced, if ever?

This RFC attempts to answer that question.

This is a partial-rewrite of [the epochs RFC]. Differences are **[marked]**.

[stability without stagnation]: https://blog.rust-lang.org/2014/10/30/Stability.html
[the epochs RFC]: https://github.com/aturon/rfcs/blob/epochs/text/0000-epochs.md

## Subject

This RFC concerns versioning and stability of the **Rust language**, encompassing the
language itself, the core libraries, and documentation of these.

The RFC also touches on versioning of the **Rust toolset**, comprising rustc, Cargo, and
related tools [?].


# Motivation
[motivation]: #motivation

**[New text]**
It is important to bear in mind that there are multiple reasons to care about stability:

*   Annie found some neglected Rust code apparently solving her problem, and just wants to run
    it with as little fuss as possible.
*   Bert is building a flashy new GitHub CLI tool, but isn't a fan of frequently changing
    specifications, and wants to specify a Rust version then not be bothered by warnings about
    features deprecated by the next major version.
*   Chris, on the other hand, wants the latest-and-greatest, and wants the project prepared in
    advance so that the version number can be updated the moment the next Rust is stable.
*   Dave is building an app for Windows 23. Unfortunately the Rust tool-chain shipping with Win 23
    is already 4 years out-of-date, so he needs to target an old version; to complicate matters,
    he also wants his app to compile on the latest Ubuntu release with no fuss.

## The status quo

Today, Rust evolution happens steadily through a combination of several mechanisms:

- **The nightly/stable release channel split**. Features that are still under
  development are usable *only* on the nightly channel, preventing *de facto*
  lock-in and thus leaving us free to iterate in ways that involve code breakage
  before "stabilizing" the feature.

- **The rapid (six week) release process**. Frequent releases on the stable
  channel allow features to stabilize as they become ready, rather than as part
  of a massive push toward an infrequent "feature-based" release. Consequently,
  Rust evolves in steady, small increments.

- **Deprecation**. Compiler support for deprecating language features and
  library APIs makes it possible to nudge people toward newer idioms without
  breaking existing code.

All told, the tools work together quite nicely to allow Rust to change and grow
over time, while keeping old code working (with only occasional, very minor
adjustments to account for things like changes to type inference.)

## What's missing

So, what's the problem?

There are two desires that the current process doesn't have a good story for:

- **Changes that may require some breakage in corner cases**. The simplest
  example is adding new keywords: the current implementation of `catch` uses the
  syntax `do catch` because `catch` is not a keyword, and cannot be added even
  as a contextual keyword without potential breakage. There are plenty of
  examples of "superficial" breakage like this that do not fit into the current
  evolution mechanisms.

- **Lack of clear "chapters" in the evolutionary story**. A downside to rapid
  releases is that, while the constant small changes eventually add up to large
  shifts in idioms, there's not an agreed upon line of demarcation between these
  major shifts. That is, we don't have a simple way to talk about these shifts,
  nor to explain the "big steps" Rust is taking when we talk to those outside
  the Rust community. If you think about the combination of `?` syntax, ATCs,
  `impl Trait`, and specialization all becoming available, for example, it's
  helpful to have an umbrella moniker like "Rust 2018" to refer to that
  encompasses them all, and the new idioms they lead to.

At the same time, the commitment to stability and rapid releases has been an
incredible boon for Rust, and we don't want to give up those existing mechanisms.

**[Changed]**
This RFC proposes requirements on what may change between major Rust versions,
how stability with old versions is maintained, and how old code is updated for new releases.

# Detailed design
[design]: #detailed-design

## The basic idea

**[Significant changes]**

- A major version series for the Rust language represents a multi-year accumulation
  of features, improvements, and idiom shifts for Rust.

- Minor version numbers continue to introduce new features, but may not introduce
  breaking changes.

- Each **crate** declares the required major and optionally minor language version
  in its `Cargo.toml`, e.g. `rust_version = "2.0"`. The major version must be matched
  exactly, while the minor version is considered to be minimum requirement of the
  crate. If the version is not specified, `1.0` is assumed (thus ensuring backwards
  compatibility and making new major versions opt-in.)

- Rust toolsets *should* support all historical major language versions, specified
  via a command-line option, and again defaulting to `1` (Rust 1.0).

New major versions are only allowed to introduce breaking changes via *deprecations*.
These work slightly differently to the way they do now:

- Deprecation lints continue to be introduced by new minor versions, but are not enabled
  by default. (This is important, since e.g. Bert wants to develop to a fixed language
  specification and Dave may be forced to use features already deprecated by the latest
  language version.)
- Deprecation lints are enabled on an opt-in basis via a command-line option
- New major language versions may turn deprecations into hard errors. This is the only kind
  of change a new major version can make beyond what a minor version can do.

Additionally,

- Features introduced after a major version increment are not automatically made available
  to the previous major version; however, new minor releases for the old major version
  may continue to be released (like Python 2.6 and 2.7 were released after 3.0).
- Exceptionally, if a security issue is found which cannot be fixed without API change,
  the corresponding deprecation lint will be enabled by default as a security warning,
  and not disabled by any catch-all disable-deprecations option.

Code that compiles without warnings on version x.y should continue to compile without
warnings on all versions x.z for z > y, modulo the [usual caveats] about type inference
changes and so on. Updating to the next major version will turn deprecations into hard
errors; alternatively the deprecations may be enabled as warnings while still using
a previous major version. Furthermore, new toolsets *should* support old major versions
indefinitely, although obviously this cannot be enforced for all compilers.

[usual caveats]: https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md

## Toolsets and versions

**[new section]**

Today, new Rust language features are developed in concert with new toolchain versions.
(Roughly speaking,) an RFC is written, an implementation is written in Rustc and exposed under a
feature flag in nightly releases, the proposed change is reviewed, and, if accepted, the feature
becomes available in the next stable toolchain.

This works well today, but it may not always be desirable to fix language and toolchain
versions, and the existing versioning becomes confusing in the context of other compilers
and new major language versions.

It is proposed that the existing Rust version numbers are inherited by the language;
the toolchain may continue incrementing versions in lock-step, but it must be clear that
a 2.x toolchain will support both 1.z and 2.x language versions, for some z (possibly
larger than the last 1.* toolchain version).

## Version timing, stabilizations, and the roadmap process

**[minor changes]**

While this RFC will focus largely on the mechanics around deprecation, a key
point is that major versions also give a *name* to large idiom shifts, recognizing that
a significant set of new features have stabilized and should change
the way you write code and construct APIs.

The proposal is much akin to the versioning used by other languages: e.g. Java and
Python, or C++ standards but with different names (e.g. C++11, C++14).
In each case, either separate compilers are maintained for each language version,
or compilers generally take flags to let you select *which* standard you wish to
compile against. It is required that Rust toolchains take the latter approach
(support for multiple versions in the same compiler) for compatibility between
crates using different versions (see below).

**[unchanged; applicable but with terminology challenges]**

To some degree, you can understand these as "marketing" releases; they bundle
together a set of changes into a (hopefully coherent) package that's easy to
talk about. But the benefits go beyond marketing: for example, it becomes much
easier for a blog post to give context about *which era* of the language you're
using without specifying a particular compiler version.

In the Rust world, we want to layer this kind of narrative on top of our
existing release and roadmap process:

- As today, each year has a [roadmap setting out that year's vision]. Some
  years---like 2017---the roadmap is mostly about laying down major new
  groundwork. Some years, however, the roadmap explicitly proposes to declare a
  new epoch during the year.

- Epoch years are focused primarily on *stabilization* and *polish*. We are
  trying to put together and ship a coherent product, complete with
  documentation and a well-aligned ecosystem. These goals will provide a
  rallying point for the whole community, to put our best foot forward as we
  publish a significant new version of the project.

[roadmap laying out that year's vision]: https://github.com/rust-lang/rfcs/pull/1728

You can see this process graphically below:

<img src="../resources/epochs.png">

The diagram shows every other compiler release due to space constraints. The
releases in red, bold text represent the compiler supporting a new epoch.

Vitally, the rapid release process---including stabilization---continues just as
today. In particular, **epoch releases are *not* feature-based**; there is no
commitment to ship a particular set of features with the first compiler version
supporting a new epoch, and features that miss that original release will
continually have additional chances to stabilize afterward, still within that
epoch. On the other hand, when we declare an epoch year in the roadmap, we will
likely have a pretty decent chance of what's *likely* to ship as stable in the
first epoch release.

### Preview versions

**[minor changes]**

To provide the clarity discussed above, and to allow us to stabilize
improvements as they're ready, we'll introduce one more concept: *preview versions*,
denoted in the figure as green, italic compiler releases.

The problem is that for changes that rely on a new major version, such as introducing a
new keyword, we cannot stabilize them within the existing major series as-is; that
would be a breaking change. On the other hand, we *want* to stabilize them as
they become ready, rather than tying all of the stabilizations to a high-stakes
major version release. And finally, we want to bundle them all together under a new moniker.

We thread the needle by providing a *preview version* at some point prior to the
new major version being shipped: `rust_version = "1.0-next"`. This preview includes *all*
of the hard errors that will be introduced in the next major version, but not yet all of
the stabilizations. It is usable from the stable channel, although not itself truely stable
since more hard errors may be introduced before the next major version is released.

The main reason to provide such a preview:

- Most importantly, it clears the way to shipping features for the next major version on
  the stable channel as they become ready, even if they require some existing
  usages to become errors. Again, keyword introduction is a simple example: the
  `1.0-next` version can begin by making it a hard error to use `catch` as an
  identifier (which is allowed today), and then later stabilizing the new
  `catch` feature when it is ready.

**[again, terminology challenges:]**

Putting this all together, in an "epoch year" the roadmap will lay out an
expected set of deprecations-to-become-errors and potential features; early in
the year, an epoch preview will be released which makes all the slated
deprecations into errors. As the year progresses, features will continue
stabilize but some may only be available by opting into the preview epoch (since
they rely on e.g. new keywords). Finally, when we're ready to ship the full new
product, we enable the new epoch and deprecate the preview version (which will
just be an alias for it).

There are some alternative ways to achieve similar ends, but with significant
downsides; these are explored in the Alternatives section.

## Constraints on major version changes

We have already constrained changes to essentially removing deprecations
(and thus making way for new features). But we want to further constrain them,
for two reasons:

- **Limiting technical debt**. The compiler retains compatibility for old
  major versions, and thus must have distinct "modes" for dealing with them. We need to
  strongly limit the amount and complexity of code needed for these modes, or
  the compiler will become very difficult to maintain.

- **Limiting deep conceptual changes**. Just as we want to keep the compiler
  maintainable, so too do we want to keep the conceptual model sustainable. That
  is, if we make truly radical changes in a new major version, it will be very difficult
  for people to reason about code involving different major versions, or to remember the
  precise differences.

As such, the RFC proposes to limit changes to be "superficial",
i.e. occurring purely in the early front-end stages of the compiler. More
concretely:

- We identify "core Rust" as being, roughly, MIR and the core trait system.
  - Over time, we'll want to make this definition more precise, but this is best
    done in an iterative fashion rather than specified completely up front.
- Major versions can only change the front-end translation into core Rust.

Hence, **the compiler supports only a single version of core Rust**; all the
"version modes" boil down to keeping around multiple desugarings into this core
Rust, which greatly limits the complexity and technical debt involved. Similarly,
core Rust encompasses the core *conceptual* model of the language, and this
constraint guarantees that, even when working with multiple major versions, those core
concepts remain fixed.

Incidentally, these constraints also mean that major version changes should be
amenable to a `rustfix` tool that automatically, and perfectly, upgrades code to
a new major version. It's an open question whether we want to *require* such a tool
before making the release.

### What major versions can do

Given those basics, let's look in more detail at a few examples of the kinds of
changes new major versions enable. These are just examples---this RFC doesn't entail any
commitment to these language changes.

#### Example: new keywords

**[minor changes, plus warning changes]**

We've taken as a running example introducing new keywords, which sometimes
cannot be done backwards compatibly (because a contextual keyword isn't
possible). Let's see how this works out for the case of `catch`, assuming that
we're currently on major version 1.

- First, we deprecate uses of `catch` as identifiers, preparing it to become a new keyword.
- We may, as today, implement the new `catch` feature using a temporary syntax
  for nightly (like `do catch`).
- When the preview `1.0-next` is released, opting into it makes `catch` into a
  keyword, regardless of whether the `catch` feature has been implemented. This
  means that opting in may require some adjustment to your code.
- The `catch` syntax can be hooked into an implementation usable on nightly with
  the preview version.
- When we're confident in the `catch` feature on nightly, we can stabilize it
  *onto the stable channel for users opting into `1.0-next`*. It cannot be stabilized by a
  1.x minor version, since it requires a new keyword.
- At some point, version `2.0` is fully shipped, meaning that `1.0-next`
  becomes an alias for `2.0`.
- `catch` is now a part of Rust.

To make this even more concrete, let's imagine the following (aligned with the diagram above):

| Rust version | Preview version | Status of `catch` in version 1 | Status of `catch` in latest version
| ------------ | ---------------------- | -- | -- |
| 1.15 | (none) | Valid identifier | Valid identifier
| 1.21 | (none) | Valid identifier; deprecated | Valid identifier; deprecated
| 1.23 | 1.0-next | Valid identifier; deprecated | Keyword
| 2.0 | (none) | Valid identifier; deprecated | Keyword

Now, suppose you have the following code:

```
Cargo.toml:

rust_version = "1.0"
```

```rust
// main.rs:

fn main() {
    let catch = "gotcha";
    println!("{}", catch);
}
```

- This code will compile **as-is** on *all* Rust toolset versions, normally without extra warnings.

- On versions 1.21 and above, when deprecation warnings are enabled, it will yield a warning,
  saying that `catch` is deprecated as an identifier.

- On version 1.23, if you change `Cargo.toml` to use the preview `1.0-next`, the
  code will fail to compile due to `catch` being a keyword. Similarly for `2.0` after
  the 2.0 release.

- However, if you leave it at version `1.0`, you can upgrade to the Rust 2.0 toolset **and
  use libraries that opt in to the `2.0` major release** with no problem.

#### Example: repurposing corner cases

A similar story plays out for more complex modifications that repurpose existing
usages. For example, some suggested module system improvements (not yet in RFC)
deduce the module hierarchy from the filesystem. But there is a corner case
today of providing both a `lib.rs` and a `bin.rs` directly at the top level,
which doesn't play well with the new feature.

**[minor change: terminology]**
A new major version could deprecate such usage (in favor of the `bin` directory),
then make it an error on the preview version. The module system change could then
be made available (and ultimately stabilized) within the preview version, before
shipping on the next major release.

#### Example: repurposing syntax

A more radical example: changing the syntax for trait objects and `impl
Trait`. In particular, we have
sometimes [discussed](https://github.com/rust-lang/rfcs/pull/1603):

- Using `dyn Trait` for trait objects (e.g. `Box<dyn Iterator<Item = u32>>`)
- Repurposing "bare `Trait` to use instead of `impl Trait`, so you can write `fn
  foo() -> Iterator<Item = u32>` instead of `fn foo -> impl Iterator<Item =
  u32>`

**[minor change: terminology]**
Suppose we wanted to carry out such a change. We could do it over multiple steps:

- First, introduce and stabilize `dyn Trait`.
- Deprecate bare `Trait` syntax in favor of `dyn Trait`.
- In a preview release, make it an error to use bare `Trait` syntax.
- Ship the new major release, and wait until bare `Trait` syntax is obscure.
- Re-introduce bare `Trait` syntax, stabilize it, and deprecate `impl Trait` in
  favor of it.

Of course, this RFC isn't suggesting that such a course of action is a *good*
one, just that it is *possible* to do without breakage.

#### Example: type inference changes

There are a number of details about type inference that seem suboptimal:

- Currently multi-parameter traits like `AsRef<T>` will infer the value of one
  parameter on the basis of the other. We would at least like an opt-out, but
  employing it for `AsRef` is backwards-incompatible.
- Coercions don’t always trigger when we wish they would, but altering the rules
  may cause other programs to stop compiling.
- In trait selection, where-clauses take precedence over impls; changing this is backwards-incompatible.

**[minor change: terminology]**
We may or may not be able to change these details without a new major version. With
enough effort, we could probably deprecate cases where type inference rules
might change and request explicit type annotations, and then—in the new
major version—those rules.

### What major versions can't do

**[minor change: terminology]**

There are also changes that new major versions don't help with, due to the constraints we
impose. These limitations are extremely important for keeping the compiler
maintainable, the language understandable, and the ecosystem compatible.

#### Example: changes to coherence rules

Trait coherence rules, like the "orphan" rule, provide a kind of protocol about
which crates can provide which `impl`s. It's not possible to change protocol
incompatibly, because existing code will assume the current protocol and provide
impls accordingly, and there's no way to work around that fact via deprecation.

**[minor change: terminology]**
More generally, this means that major versions can only be used to make changes to the
language that are applicable *crate-locally*; they cannot impose new
requirements or semantics on external crates, since we want to retain
compatibility with the existing ecosystem.

#### Example: `Error` trait downcasting

**[minor change: terminology]**
See [rust-lang/rust#35943](https://github.com/mozilla/rust/issues/35943). Due to
a silly oversight, you can’t currently downcast the “cause” of an error to
introspect what it is. We can’t make the trait have stricter requirements; it
would break existing impls. And there's no way to do so only in a newer major version,
because we must be compatible with the older one, meaning that we cannot rely on
downcasting.

This is essentially another example of a non-crate-local change.

## The full mechanics

**[minor change: terminology]**
We'll wrap up with the full details of the mechanisms at play.

- `rustc` will take a new flag, `--rust-version`, which can specify the major version to
  use. This flag will default to 1.0.
  - This flag should not affect the behavior of the core trait system or passes at the MIR level.
- `Cargo.toml` can include an `rust_version` value, which is used to pass to `rustc`.
  - If left off, it will assume version 1.0.
- `cargo new` will produce a `Cargo.toml` with the latest major version, and minor 0.
  Preview versions will not be used by default since these are not stable.
  **[change: preview is not auto-selected]**

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

**[change of message]**

First and foremost, if we accept this RFC, we should publicize the plan widely,
including on the main Rust blog, in a style simlar to [previous posts] about our
release policy. This will require extremely careful messaging, to make clear
that previous major versions will remain available with full support, including
linking crates using different major versions, thus not risking division of the Rust
ecosystem.

In addition, the book should talk about the basics from a user perspective,
including:

- The fact that, if you do nothing, your code should continue to compile (with
  minimum hassle) when upgrading the compiler.
- If you resolve deprecations as they occur, moving to a new major version
  should also require minimum hassle.
- Best practices about upgrading major versions (TBD).

[previous posts]: https://blog.rust-lang.org/2014/10/30/Stability.html

# Drawbacks
[drawbacks]: #drawbacks

**[minor change: terminology]**

There are several drawbacks to this proposal:

- Most importantly, it risks muddying our story about stability, which we've
  worked very hard to message clearly.

  - To mitigate this, we need to put front and center that, **if you do nothing,
  updating to a new `rustc` should not be a hassle**, and **staying on an old
  major version doesn't cut you off from the ecosystem**.

- It adds a degree of complication to an evolution story that is already
  somewhat complex (with release channels and rapid releases).

  - On the other hand, major version releases can provide greater clarity about major
    steps in Rust evolution, for those who are not following development 
    closely.

- New major versions can invalidate existing blog posts and documentation, a problem we
  suffered a lot around the 1.0 release

  - However, this situation already obtains in the sense of changing idioms; a
    blog post using `try!` these days already feels like it's using "old
    Rust". Notably, though, the code still compiles on current Rust.

  - A saving grace is that, with major versions, it's much more likely that a post will
    mention which version is being used, for context. Moreover, with sufficient
    work on error messages, it seems plausible to detect that code was intended
    for an earlier major version and explain the situation.

# Alternatives
[alternatives]: #alternatives

## Within the basic epoch structure

Sticking with the basic idea of epochs, there are a couple alternative setups,
that avoid "preview" epochs.

- Rather than locking in a set of deprecations in a preview epoch, we could
  provide "stable channel feature gates", allowing users to opt in to features
  of the next epoch in a fine-grained way, which may introduce new errors.
  When the new epoch is released, one would then upgrade to it and remove all of the gates.

  - The main downside is lack of clarity about what the current "stable Rust"
    is; each combination of gates gives you a slightly different language. While
    this fine-grained variation is acceptable for nightly, since it's meant for
    experimentation, it cuts against some of the overall goals of this proposal
    to introduce such fragmentation on the stable channel.

- We could stabilize features using undesirable syntax at first, making way for
  better syntax only when the new epoch is released, then deprecate the "bad"
  syntax in favor of the "good" syntax.

  - For `catch`, this would look like:
    - Stabilize `do catch`.
    - Deprecate `catch` as an identifier.
    - Ship new epoch, which makes `catch` a keyword.
    - Stabilize `catch` as a syntax for the `catch` feature, and deprecate `do catch` in favor of it.
  - This approach involves significantly more churn than the one proposed in the RFC.

- Finally, we could just wait to stabilize features like `catch` until the
  moment the epoch is released.

  - This approach seems likely to introduce all the downsides of "feature-based"
    releases, making the epoch release extremely high stakes, and preventing
    usage of "ready to go" feature on the stable channel until the epoch is
    shipped.

## Alternatives

The larger alternatives include, of course, not trying to solve the problems
laid out in the motivation, and instead finding creative alternatives.

- For cases like `catch` that require a new keyword, it's not clear how to do
this without ending up with suboptimal syntax.

**[changed below]**

Different terminology is possible, as in [the epochs RFC], however beyond terminology
the two approaches are largely the same, save that *epochs* provide an alternative
way to differentiate *language* and *toolchain* versions, but don't allow
*minor language versions* or clarify how/if new features are made available to
old epochs.

It would of course be possible to allow greater changes in new language versions, including
breaking compatibility between crates using different versions, however this goes
against the promise of [stability without stagnation].

`rustc` could default to the latest major language version instead of the first (stable)
version, however this breaks backwards compatibility in the strict sense.

# Unresolved questions
[unresolved]: #unresolved-questions

**[minor changes; default version moved above]**

- It's not clear what the story should be for the books and other
  documentation. Should we gate shipping a new major release on having a fully updated
  book? The preview version---and the fact that everything is shipped on stable
  prior to releasing the version---would make that plausible.

- Do we want to require a `rustfix` for all deprecations enforced by new major versions?

- Will we ever consider dropping support for very old major version series? Given the
  constraints in this RFC, it seems unlikely to ever be worth it.
