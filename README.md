# inline-proc

This crate provides the ability to write procedural macros directly in your code, instead of
having to use another crate.

[Repo](https://github.com/KaiJewson/inline-proc) - [crates.io](https://crates.io/crates/inline-proc) - [documentation](https://docs.rs/inline-proc)

## Example
```rust
use inline_proc::inline_proc;

#[inline_proc]
mod example {
    metadata::ron!(
        edition: "2018",
        clippy: true,
        dependencies: {
            "quote": "1",
        },
        exports: (
            bang_macros: {
                "def_func": "define_function",
            },
        ),
    );

    pub fn define_function(_: proc_macro::TokenStream) -> proc_macro::TokenStream {
        quote::quote!(
            fn macro_function() {
                println!("Hello from a proc macro!");
            }
        ).into()
    }
}

def_func!();

macro_function();
// => Hello from a proc macro!
```

## How It Works

`inline-proc` takes your code and puts it in a crate with the path
`{temporary directory}/inline-proc-crates/{package name}-{significant package version}-{module name}`.
For example, if you call `inline-proc` on Linux from the module `my_module` in `my-nice-crate`
which has version `0.7.3`, a temporary crate will be created in
`/tmp/inline-proc-crates/my-nice-crate-0.7-my_module`.

It then compiles this crate as a `dylib` with Cargo and translates all the outputted errors into
errors from the proc macro, so it appears identical to writing the code inline. Note that proc
macros cannot currently emit warnings on stable, so you will have to use nightly if you want
that.

It outputs `macro_rules!` macros that expand to invocations of the private
`inline_proc::invoke_inline_macro!` macro. This macro takes in the path of a dylib generated by
the `inline_proc` attribute macro, the name of the macro that is inside that dylib, the type of
macro that it is (bang/derive/attribute) and the input to the macro. It opens up the dylib and
calls the macro, returning its result.

## Using the generated macros

The macros generated by `#[inline_proc]` can be used directly:

```rust
use inline_proc::inline_proc;

#[inline_proc]
mod direct_usage {
    metadata::ron!(
        edition: "2018",
        dependencies: {},
        exports: (
            bang_macros: { "my_bang_macro": "my_bang_macro" },
            derives: { "MyDeriveMacro": "my_derive_macro" },
            attributes: { "my_attribute_macro": "my_attribute_macro" },
        ),
    );
    use proc_macro::TokenStream;

    pub fn my_bang_macro(_input: TokenStream) -> TokenStream {
        todo!()
    }
    pub fn my_derive_macro(_item: TokenStream) -> TokenStream {
        todo!()
    }
    pub fn my_attribute_macro(_attr: TokenStream, _item: TokenStream) -> TokenStream {
        todo!()
    }
}

my_bang_macro!(input tokens);
MyDeriveMacro!(struct InnerItem;);
my_attribute_macro!((attribute tokens) struct InnerItem;);
```

This works fine for bang macros, but is not so good for attribute or derive macros. So this
crate provides the attribute and derive macros `#[inline_attr]` and `InlineDerive`; they can be
used like this:

```rust
#
use inline_proc::{InlineDerive, inline_attr};

#[derive(InlineDerive)]
#[inline_derive(MyDeriveMacro)]
struct InnerItem;

#[inline_attr[my_attribute_macro(attribute tokens)]]
struct InnerItem;
```

They expand to the same code as above.

## Exporting the macros

In order to export your macro, you will first have to change your macro definition to:
`"macro_name": ( function: "macro_function", export: true )`. This will do three things:

1. Label the generated `macro_rules!` with `#[macro_export]` and `#[doc(hidden)]`.
1. Have the macro take a path to `inline_proc::invoke_inline_macro`.
1. Suffix the macro's name with `_inner`.

You then create a wrapper around it like so:

```rust
// At the crate root

#[doc(hidden)]
pub use inline_proc::invoke_inline_macro;

// Where your #[inline_proc] is

/// This macro does XYZ.
#[macro_export]
macro_rules! my_macro {
    ($($tt:tt)*) => {
        $crate::my_macro_inner!($crate::invoke_inline_macro, $($tt)*);
    }
}
```

This level of indirection is necessary as proc macros don't have a way of getting the current
crate like MBEs do (`$crate`), so you have to supply it via the MBE method.

## Caveats

This approach comes with several caveats over regular proc macros:
- Slower compilation speeds as a second Cargo instance has to be invoked.
- Not able to use TOML to define dependencies.
- Exporting macros is a pain.
- The macros can only be defined in one file.
- Errors are a lot less helpful. This is improved a bit by Nightly, but still isn't is good as
native proc macro errors.
- Derive helper attributes are not supported. The `InlineDerive` macro does reserve the `helper`
helper attribute, so you can for example replace `#[my_helper]` with `#[helper[my_helper]]`.

License: MIT OR Apache-2.0
