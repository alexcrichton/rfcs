- Feature Name: `rustc_macros`
- Start Date: 2016-07-14
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Stabilize a very small sliver of today's procedural macro system in the
compiler, just enough to get basic features like custom derive working. Ensure
that the features stabilized here will not pose a maintenance burden on the
compiler but also don't try to stabilize enough features for the "perfect macro
system" at the same time. Overall, this should be considered an incremental
step towards an official "macros 2.0".

# Motivation
[motivation]: #motivation

Some large projects in the ecosystem today, such as [serde] and [diesel],
effectively require the nightly channel of the Rust compiler. Although most
projects have an alternative to work on stable Rust, this tends to be far less
ergonomic and comes with its own set of downsides, and empirically it has not
been enough to push the nightly users to stable as well.

[serde]: https://github.com/serde-rs/serde
[diesel]: http://diesel.rs/

These large projects, however, are often the face of Rust to external users.
Common knowledge is that fast serialization is done using serde, but to others
this just sounds likes "fast Rust needs nightly". Over time this persistent
thought process creates a culture of "well to be serious you require nightly"
and a general feeling that Rust is not "production ready".

The good news, however, is that this class of projects which require nightly
Rust almost all require nightly for the reason of procedural macros. Even
better, the full functionality of procedural macros is rarely needed, only
custom derive! Even better, custom derive typically doesn't *require* the features
one would expect from a full-on macro system, such as hygiene and modularity,
that normal procedural macros typically do. The purpose of this RFC, as a
result, is to provide these crates a method of working on stable Rust with the
desired ergonomics one would have on nightly otherwise.

Unfortunately today's procedural macros are not without their architectural
shortcomings as well. For example they're defined and imported with arcane
syntax and don't participate in hygiene very well. To address these issues,
there are a number of RFCs to develop a "macros 2.0" story:

* [Changes to name resolution](https://github.com/rust-lang/rfcs/pull/1560)
* [Macro naming and modularisation](https://github.com/rust-lang/rfcs/pull/1561)
* [Procedural macros](https://github.com/rust-lang/rfcs/pull/1566)
* [Macros by example 2.0](https://github.com/rust-lang/rfcs/pull/1584)

Many of these designs, however, will require a significant amount of work to not
only implement but also a significant amount of work to stabilize. The current
understanding is that these improvements are on the time scale of years, whereas
the problem of nightly Rust is today!

As a result, it is an explicit non-goal of this RFC to architecturally improve
on the current procedural macro system. The drawbacks of today's procedural
macros will be the same as those proposed in this RFC. The major goal here is
to simply minimize the exposed surface area between procedural macros and the
compiler to ensure that the interface is well defined and can be stably
implemented in future versions of the compiler as well.

Put another way, we currently have macros 1.0 unstable today, we're shooting
for macros 2.0 stable in the far future, but this RFC is striking a middle
ground at macros 1.1 today!

# Detailed design
[design]: #detailed-design

First, before looking how we're going to stabilize procedural macros, let's
take a detailed look at how they work today.

### Today's procedural macros

A procedural macro today is loaded into a crate with the `#![plugin(foo)]`
annotation at the crate root. This in turn looks for a crate named `foo` [via
the same crate loading mechanisms][loader] as `extern crate`, except [with the
restriction][host-restriction] that the target triple of the crate must be the
same as the target the compiler was compiled for. In other words, if you're on
x86 compiling to ARM, macros must also be compiled for x86.

[loader]: https://github.com/rust-lang/rust/blob/78d49bfac2bbcd48de522199212a1209f498e834/src/librustc_metadata/creader.rs#L480
[host-restriction]: https://github.com/rust-lang/rust/blob/78d49bfac2bbcd48de522199212a1209f498e834/src/librustc_metadata/creader.rs#L494

Once a crate is found, it's required to be a dynamic library as well, and once
that's all verified the compiler [opens it up with `dlopen`][dlopen] (or the
equivalent therein). After loading, the compiler will [look for a special
symbol][symbol] in the dynamic library, and then call it with a macro context.

[dlopen]: https://github.com/rust-lang/rust/blob/78d49bfac2bbcd48de522199212a1209f498e834/src/librustc_plugin/load.rs#L124
[symbol]: https://github.com/rust-lang/rust/blob/78d49bfac2bbcd48de522199212a1209f498e834/src/librustc_plugin/load.rs#L136-L139

So as we've seen macros are compiled as normal crates into dynamic libraries.
One function in the crate is tagged with `#[plugin_registrar]` which gets wired
up to this "special symbol" the compiler wants. When the function is called with
a macro context, it uses the passed in [plugin registry][registry] to register
custom macros, attributes, etc.

[registry]: https://github.com/rust-lang/rust/blob/78d49bfac2bbcd48de522199212a1209f498e834/src/librustc_plugin/registry.rs#L30-L69

After a macro is registered, the compiler will then continue the normal process
of expanding a crate. Whenever the compiler encounters this macro it will call
this registration with essentially and AST and morally gets back a different
AST to splice in or replace.

### Today's drawbacks

This expansion process suffers from many of the downsides mentioned in the
motiviation section, such as a lack of hygiene, a lack of modularity, and the
inability to import macros as you would normally other functionality in the
module system.

Additionally, though, it's essentially impossible to ever *stabilize* because
the interface to the compiler is... the compiler! We clearly want to make
changes to the compiler over time, so this isn't acceptable. To have a stable
interface we'll need to cut down this surface area *dramatically* to a curated
set of known-stable APIs.

Somewhat more subtly, the technical ABI of procedural macros is also exposed
quite thinly today as well. The implementation detail of dynamic libraries, and
especially that both the compiler and the macro dynamically link to libraries
like libsyntax, cannot be changed. This precludes, for example, a completely
statically linked compiler (e.g. compiled for `x86_64-unknown-linux-musl`).
Another goal of this RFC will also be to hide as many of these technical
details as possible, allowing the compiler to flexibly change how it interfaces
to macros.

## Macros 1.1

Ok, with the background knowledge of what procedural macros are today, let's
take a look at how we can solve the major problems blocking its stabilization:

* Sharing an API of the entire compiler
* Frozen interface between the compiler and macros

### `libmacro`

Proposed in [RFC 1566](https://github.com/rust-lang/rfcs/pull/1566) and
described in [this blog post](http://ncameron.org/blog/libmacro/) the
distribution will now ship with a new `libmacro` crate available for macro
authors. The intention here is that the gory details of how macros *actually*
talk to the compiler is entirely contained within this one crate. The stable
interface to the compiler is then entirely defined in this crate, and we can
make it as small or large as we want. Additionally, like the standard library,
it can contain unstable APIs to test out new pieces of functionality over time.

The initial implementation of `libmacro` is proposed to be *incredibly* bare
bones:

```rust
#![crate_name = "macro"]

pub struct TokenStream {
    // ...
}

#[derive(Debug)]
pub struct LexError {
    // ...
}

pub struct MacroContext {
    // ...
}

impl TokenStream {
    pub fn from_source(cx: &mut MacroContext,
                        source: &str) -> Result<TokenStream, LexError> {
        // ...
    }

    pub fn to_source(&self, cx: &mut MacroContext) -> String {
        // ...
    }
}
```

That is, there will only be a handful of exposed types and `TokenStream` can
only be converted to and from a `String`. Eventually `TokenStream` type will
more closely resemble token streams [in the compiler
itself][compiler-tokenstream], and more fine-grained manipulations will be
available as well.

Additionally, the `MacroContext` structure will initially be completely devoid
of functionality, but in the future it will be the entry point for [many other
features][macro20] one would expect in macros 2.0

[compiler-tokenstream]: https://github.com/rust-lang/rust/blob/master/src/libsyntax/tokenstream.rs#L323-L338
[macro20]: http://ncameron.org/blog/libmacro/

### Defining a macro

A new crate type will be added to the compiler, `rustc-macro`, indicating a
crate that's compiled as a procedural macro. Like the executable, staticlib,
and cdylib crate types the `rustc-macro` crate type is intended to be a final
product.  What it *actually* produces is not specified, but if a `-L` path is
provided to it then the compiler will recognize the output artifacts as a
macro and it can be loaded for a program.

Unlike today, however, there will not be a "registrar" function, but rather a
number of functions which act as token stream tranformers to implement macro
functionality.

Putting this together, a macro crate might look like:

```rust
#![crate_type = "rustc-macro"]
#![crate_name = "double"]

extern crate rustc_macro;

use rustc_macro::{MacroContext, TokenStream};

// Rough equivalent of:
//
// ```
// macro_rules! double {
//     ($e:expr) => (2 * $e)
// }
// ```
//
// but requires `$e` to be a literal integer
#[rustc_macro_define(double)]
pub fn double(cx: &mut MacroContext, input: TokenStream) -> TokenStream {
    let input = input.to_source(cx);
    let input = input.parse::<u64>().unwrap();
    let output = (2 * input).to_string();
    let output = TokenStream::from_source(cx, &output).unwrap();
    return output
}

#[rustc_macro_derive(Double)]
pub fn double(cx: &mut MacroContext, input: TokenStream) -> TokenStream {
    // Convert `input` to a string, parse a struct/enum declaration, and then
    // return back source representing a number of items representing the
    // implementation of the `Double` trait for the struct/enum in question.

    // ...
}
```

Two new attributes will be allowed inside of a `rustc-macro` crate (disallowed
in other crate types):

* `rustc_macro_define` - defines a new `macro!`-style macro to be defined by
  this crate. The input to the macro is everything between the macro
  delimeters, and the output is the source that the macro will expand to. Note
  that no hygiene happens here, the source is simply copy/pasted. Additionally,
  if an invocation like `double!(foo!())` is encountered, `foo!()` *will not be
  expanded* ahead of time.

* `rustc_macro_derive` - defines a new `#[derive]` mode which can be used in a
  crate. The input here is the entire struct that `#[derive]` was attached to,
  attributes and all. The output is **expected to include the
  `struct`/`enum` itself**, as well as any number of items to be contextually
  "placed next to" the initial declaration. Again, though, there is no hygiene,
  it's as if the source was simply copy/pasted.

Each `rustc_macro_*` attribute requires the signature (similar to [macros
2.0][mac20sig]):

[mac20sig]: http://ncameron.org/blog/libmacro/#tokenisingandquasiquoting

```rust
fn(&mut MacroContext, TokenStream) -> TokenStream
```

If a macro cannot process the input token stream, it is expected to panic for
now, although eventually it will call methods on `MacroContext` to provide more
structured errors. The compiler will wrap up the panic message and display it
to the user appropriately. Eventually, however, libmacro will provide more
interesting methods of signaling errors to users.

Customization of user-defined `#[derive]` modes can still be done through custom
attributes, although it will be required for `rustc_macro_derive`
implementations to remove these attributes when handing them back to the
compiler. The compiler will still gate unknown attributes by default.

### Using a procedural macro

Using a procedural macro will be very similar to today's procedural macro
system, such as:

```rust
#![rustc_macro_crate(double)]

#[derive(Double)]
pub struct Foo;

fn main() {
    println!("{}", double!(2));
}
```

That is, the `#![rustc_macro_crate]` attribute, required at the crate root,
will search for a crate compiled with the `rustc-macro` crate type. If found,
the crate will be loaded (not specified how) and the derive modes and macros
will be available to the entire crate.

For now, only one argument to `#![rustc_macro_crate]` will be allowed. Macros
and derive modes will shadow one another, with the last-mentioned
`#![rustc_macro_crate]` attribute taking precedence.

### Initial implementation details

This section lays out what the initial implementation details of macros 1.1
will look like, but none of this will be specified as a stable interface to the
compiler. These exact details are subject to change over time as the
requirements of the compiler change, and even amongst platforms these details
may be subtly different.

The compiler will essentially consider `rustc-macro` crates as `--crate-type
dylib -C prefer-dyanmic`. That is, compiled the same way they are today. This
namely means that these macros  will dynamically link to the same standard
library as the compiler itself, therefore sharing resources like a global
allocator, etc.

The `libmacro` crate will compiled as an rlib and a static copy of it will be
included in each macro. This crate will provide a symbol known by the compiler
that can be dynamically loaded. The compiler will `dlopen` a macro crate in the
same way it does today, find this symbol in `libmacro`, and call it.

The `rustc_macro_define` and `rustc_macro_derive` attributes will be encoded
into the crate's metadata, and the compiler will discover all these functions,
load their function pointers, and pass them to the `libmacro` entry point as
well. This provides the opportunity to register all the various expansion
mechanisms with the compiler.

The actual underlying representation of `TokenStream` will be basically the same
as it is in the compiler today. (the details on this are a little light
intentionally, shouldn't be much need to go into *too* much detail).

## Pieces to stabilize

Eventually this RFC is intended to be considered for stabilization (after it's
implemented and proven out on nightly, of course). The summary of pieces that
would become stable are:

* The `rustc_macro` crate, and a small set of APIs within (skeleton above)
* The `rustc-macro` crate type
* The `#[rustc_macro_derive]` attribute
* The `#[rustc_macro_define]` attribute (optional)
* The signature of the `#![rustc_macro_{define,derive}]` functions
* The `#![rustc_macro_crate]` attribute
* Semantically being able to load macro crates compiled as `rustc-macro` into
  the compiler, requiring that the crate was compiled by the exact compiler.
* The semantic behavior of macros today, including:
  * Derive modes and macros are not imported, they're just dumped into the crate
    root.
  * Lack of hygiene for both
  * Shadowing behavior if you load multiple macro crates

### Macros 1.1 in practice

Alright, that's a lot to take in! Let's take a look at what this is all going to
look like in practice, focusing on a case study of `#[derive(Serialize)]` for
serde.

First off, serde will provide a crate, let's call it `serde_macros`. The
`Cargo.toml` will look like:

```toml
[package]
name = "serde-macros"
# ...

[lib]
rustc-macro = true

[dependencies]
# ...
```

The contents will look similar to

```rust
extern crate macro;
extern crate syntex_syntax;

use macro::{MacroContext, TokenStream};

#[rustc_macro_derive(Serialize)]
pub fn derive_serialize(_cx: &mut MacroContext,
                        input: TokenStream) -> TokenStream {
    let input = input.to_source();

    // use syntex_syntax from crates.io to parse `input` into an AST

    // use this AST to generate an impl of the `Serialize` trait for the type in
    // question

    // convert that impl to a string

    // parse back into a token stream
    return TokenStream::from_source(&impl_source).unwrap()
}
```

Next, crates will depend on this such as:

```toml
[dependencies]
serde-macros = "0.9"
serde = "0.9"
```

And finally use it as such:

```rust
#![rustc_macro_crate(serde_macros)]
extern crate serde;

#[derive(Serialize)]
pub struct Foo {
    a: usize,
    #[serde(rename = "foo")]
    b: String,
}
```

# Drawbacks
[drawbacks]: #drawbacks

* This is not an interface that anyone would wish to stabilize in a void, there
  are a number of known drawbacks to the current macro system in terms of how
  it architecturally fits into the compiler. Additionally, there's work underway
  to solve all these problems with macros 2.0.

  As mentioned before, however, the stable version of macros 2.0 is currently
  quite far off, and the desire for features like custom derive are very real
  today. The rationale behind this RFC is that the downsides are an acceptable
  tradeoff from moving a significat portion of the nightly ecosystem onto stable
  Rust.

* This implementation is likely to be less performant than procedural macros
  are today. Round tripping through strings isn't always a speedy operation,
  especially for larger expantions. Strings, however, are a very small
  implementation detail that's easy to see stabilized until the end of time.
  Additionally, it's planned to extend the `TokenStream` API in the future to
  allow more fine-grained transformations without having to round trip through
  strings.

* Users will still have an inferior experience to today's nightly macros
  specifically with respect to compile times. The `syntex_syntax` crate takes
  quite a few seconds to compile, and this would be required by any crate which
  uses serde. To offset this, though, the `syntex_syntax` could be *massively*
  stripped down as all it needs to do is parse struct declarations mostly. There
  are likely many other various optimizations to compile time that can be
  applied to ensure that it compiles quickly.

# Alternatives
[alternatives]: #alternatives

* Wait for macros 2.0, but this likely comes with the high cost of postponing a
  stable custom-derive experience on the time scale of years.

* Don't stabilize `rustc_macro` as a new crate, but rather specify that
  `#[rustc_macro_derive]` has a stable-ABI friendly signature. This does not
  account, however, for the eventual planned introduction of the `rustc_macro`
  crate and is significantly harder to write. The marginal benefit of being
  slightly more flexible about how it's run likely isn't worth it.

* The syntax for defining a macro may be different in the macros 2.0 world (e.g.
  `pub macro foo` vs an attribute), that is it probably won't involve a function
  attribute like `#[rustc_macro_derive]`. This interim system could possibly use
  this syntax as well, but it's unclear whether we have a concrete enough idea
  in mind to implement today.

* Instead of passing around `&mut MacroContext` we could allow for storage of
  compiler data structures in thread-local-storage. This would avoid threading
  around an extra parameter and perhaps wouldn't lose too much flexibility.

* Instead of `#![rustc_macro_crate]` macro crates could be loaded with `extern
  crate foo` instead. The downsides of this are that the compiler would have to
  generate an error if you import paths from it, so it doesn't behave like other
  `extern crate` statements, and the compiler also doesn't know whether to look
  for a target or host crate when it encounters such a definition. If both are
  found it doesn't know which is the right to choose as well.

# Unresolved questions
[unresolved]: #unresolved-questions

* Is the interface between macros and the compiler actually general enough to
  be implemented differently one day?

* The intention of macros 1.1 is to be *as close as possible* to macros 2.0 in
  spirit and implementation, just without stabilizing vast quantities of
  features. In that sense, it is the intention that given a stable macros 1.1,
  we can layer on features backwards-compatibly to get to macros 2.0. Right now,
  though, the delta between what this RFC proposes and where we'd like to is
  very small, and can get get it down to actually zero?
