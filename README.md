# `ritter` - tree-sitter grammars via rust macros

`ritter` makes it easy to create efficient parsers in Rust by leveraging the [tree-sitter](https://tree-sitter.github.io/tree-sitter/) parser generator. With `ritter`, you can define your entire grammar with proc-macro annotations on idiomatic<sup>*</sup> Rust code.

[^*]: Sort of, future plans may improve this.

## Usage

Add `ritter` runtime to your `Cargo.toml` and the build tool as a build dependency:
```toml
[dependencies]
ritter = "0.1.0-pre.1"

[build-dependencies]
ritter-tool = "0.1.0-pre.1"
```

The first step is to configure your `build.rs` to compile and link the generated tree-sitter parser:

```rust
use std::path::PathBuf;

fn main() {
    println!("cargo:rerun-if-changed=src");
    // Path to the file containing your grammar and any submodules.
    ritter_tool::build_parsers("src/grammar/mod.rs"));
}
```

## Defining a Grammar
See [examples](examples/) for some complete examples of how to define and use a grammar.

## Type Annotations
ritter supports a number of annotations that can be applied to type and fields in your grammar. These annotations can be used to control how the parser behaves, and how the resulting AST is constructed.

### `#[language]`
This annotation marks the entrypoint for parsing, and determines which AST type will be returned from parsing. Only one type in the grammar can be marked as the entrypoint.

```rust
#[derive(Rule)]
#[language]
struct Code {
    ...
}
````

### `#[extras(...)]`
This annotation can be used on the `#[language]` rule to specify a list of extras. These extras are specified
using the same DSL as `#[leaf(...)]` and `#[text(...)]`. These rules are inserted to the `extras` array in the
grammar.

```rust
#[derive(Rule)]
#[language]
#[extras(
    re(r"\s") // allows whitespace in the grammar.
)]
struct Code {
    ...
}
```

## Field Annotations
### `#[leaf(...)]` and `#[text(...)]`
The `#[leaf(...)]` annotation can be used to define a leaf node in the AST.
`#[text(...)]` is similar, but it does not create a named node in the grammar and cannot be
extracted. It must always be assigned to `()`.

`leaf` and `text` take an input that looks like the [tree sitter DSL](https://tree-sitter.github.io/tree-sitter/creating-parsers/2-the-grammar-dsl.html). The supported rules
currently are:
* `choice`
* `optional`
* `seq`
* `re` or `pattern` to specify a regular expression
* literal text

Others can be added in the future as needed.

`leaf` can either be applied to a field in a struct / enum variant (as seen above), or directly on a type with no fields:

```rust
#[derive(Rule)]
#[leaf("9")]
struct BigDigit;

#[derive(Rule)]
enum SmallDigit {
    #[leaf("0")]
    Zero,
    #[leaf("1")]
    One,
}
```

### `#[prec(...)]` / `#[prec_left(...)]` / `#[prec_right(...)]` / `#[prec_dynamic(...)]`
This annotation can be used to define a non/left/right-associative operator. This annotation takes a single parameter, which is the precedence level of the operator (higher binds more tightly).

### `#[immediate]`
Usually, whitespace is optional before each token. This attribute means that the token will only match if there is no whitespace.

### `#[skip(...)]`
This annotation can be used to define a field that does not correspond to anything in the input string, such as some metadata. This annotation takes a single parameter, which is the value that should be used to populate that field at runtime.

### `#[word]`
This annotation marks the field as a Tree Sitter [word](https://tree-sitter.github.io/tree-sitter/creating-parsers#keywords), which is useful when handling errors involving keywords. Like `#[extras]`, the `#[word]` is specified on the `#[language]` implementation:

```rust
#[derive(Debug, Rule)]
#[language]
#[word(Ident)]
pub struct Language {
    // ...
}

#[derive(Rule)]
#[leaf(re(r"[a-zA-Z_]+"))]
pub struct Ident;
```

## Partial AST and Errors
ritter, like tree-sitter, can produce a partial AST along with its errors. Calling `Language::parse` will
produce a `ParseResult` object which includes as much of the AST as it was able to extract, as well as a `Vec`
of all of the parsing errors encountered. This is useful for language servers and other contexts which can
make use of a partial AST. Currently this may not produce the _maximal_ AST, but this may be possible
in the future.

## Special Types
ritter has a few special types that can be used to define more complex grammars.

### `Vec<T>`
To parse repeating structures, you can use a `Vec<T>` to parse a list of `T`s. Note that the `Vec<T>` type **cannot** be wrapped in another `Vec` (create additional structs if this is necessary). There are two special attributes that can be applied to a `Vec` field to control the parsing behavior.

The `#[sep_by(...)]` attribute can be used to specify a separator between elements of the
list. This is parsed in the same way as `text` and `leaf` and therefore supports all of the listed tree-sitter
grammar above.

```rust
pub struct CommaSeparatedExprs {
    #[sep_by(",")]
    numbers: Vec<Expr>,
}
```

The `#[repeat1]` can be used to specify that the list must contain at least, or you can use `#[sep_by1(...)]

```rust
pub struct CommaSeparatedExprs {
    #[repeat1]
    #[sep_by(",")]
    // Or just use #[sep_by1(",")]
    numbers: Vec<Expr>,
}
```

### `Option<T>`
To parse optional structures, you can use an `Option<T>` to parse a single `T` or nothing. Like `Vec`, the `Option<T>` type **cannot** be wrapped in another `Option` (create additional structs if this is necessary). For example, we can make the list elements in the previous example optional so we can parse strings like `1,,2`:

```rust
pub struct CommaSeparatedExprs {
    #[sep_by1(",")]
    numbers: Vec<Option<Expr>>,
}
```

### `ritter::Spanned<T>`
When using ritter to power diagnostic tools, it can be helpful to access spans marking the sections of text corresponding to a parsed node. To do this, you can use the `Spanned<T>` type, which captures the underlying parsed `T` and a pair of indices for the start (inclusive) and end (exclusive) of the corresponding substring. `Spanned` types can be used anywhere, and do not affect the parsing logic. For example, we could capture the spans of the expressions in our previous example:

```rust
pub struct CommaSeparatedExprs {
    #[sep_by1(",")]
    numbers: Vec<Option<Spanned<Expr>>>,
}
```

### `Box<T>`
Boxes are automatically constructed around the inner type when parsing, but ritter doesn't do anything extra beyond that.

## Credits
Special thanks to [rust-sitter](https://github.com/hydro-project/rust-sitter). This project began as a fork of `rust-sitter`, and has since heavily diverged.
