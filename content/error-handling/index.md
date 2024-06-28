+++
title = "My incremental approach to error handling in Rust"
date = 2018-08-13

[taxonomies]
categories = ["programming", "rust"]
+++

I have recently begun working on a [small language](https://github.com/bIgBV/lispy-rs) following [this](http://www.buildyourownlisp.com) book. Instead of writing the language in C,
I chose to write in Rust. I had already written non-trivial programs in Rust
before and used this as an opportunity to get more practice building systems
in the language. When I started working on the project, my error handling
situation wasn't the best. I had a custom Error type, an enum called
`LispyError` which had a few variants which I added as I went. The error type
was then used with a corresponding result type `EvalResult`, since I thought
that the errors would have to be reported when the code would be evaluated.
This system did not scale very well as I soon realized that in order to have
good insight why your code is broken, your programming language's tools need
to point out what's wrong and present that information in an understandable
manner.

This blog post talks about how I ended up leveraging Rust's type system to
encode logic to help make the Rust compiler help me write better code.

I started by first moving my custom error and result types into their own module:

```rs
// error.rs

#[derive(Debug)]
pub enum LispyError {
    BadOp,
    BadNum,
    BadOperand,
    ListError(String),
}

impl fmt::Display for LispyError {
 // Display implementation for LispyError
}

impl Error for LispyError {}

pub type EvalResult<T> = Result<T, LispyError>;
```

Since the errors reported to the user need to be message giving information
about what happened, I changed he error type into a struct with two fields:
`kind` and `message`. Then I made a simple constructor for the error type.

```rs
struct LispyError {
    kind: ErrorKind,
    message: &'static str,
}

pub fn make_error(kind: ErrorKind, message: &str) -> LispyError {
    LispyError { kind, message }
}

```

As I started using this across the different parts of the system, I quickly
realized that this lead to a lot of code duplication. This is because most of
the errors that my system currently reported were generally of the form where
the system expected one thing and got something else. This lead to a lot of
repetition especially in my builtin methods. This is where I learnt how awesome
traits, generics and trait bounds were. All my builtin methods were created by
implementing a custom trait called `Operate`.

```rs
pub trait Operate {
    fn operate(&self, operands: &Vec<Expr>, env: &mut Env) -> EvalResult<Expr>;
}
```

This defined the logic for the operation of the given buitin. Since all of them
were using the same trait, I defined a method in the trait with a default
implementation called `handle_error`. This method accepted a kind, an expected
value and the value which the calling method called. It matched on the error
kind and returned an appropriately formatted error. The expected and got
parameters were generics with the trait bound `Debug`. This meant that as long
as the values being passed to the function implemented the `Debug` trait, the
error would be properly formatted, this also meant that at compile time, I
would be notified if I had mistakenly passed in a parameter which did not
implement the trait.

```rs
fn handle_error<G, X>(kind: ErrorKind, got: G, expected: X) where G: Display, X: Display -> LispyError {
  let fomat_string = String::from("");
  match kind {
      ErrorKind::BadArgument => format_string.push_str(
          "Funciton '{}' passed incorrect number of arguments.  Got {}, Expected: {}",
      ),
      ErrorKind::BadType => format_string.push_str(
          "Function '{}' passed incorrect type for argument 0.  Got {}, Expected {}",
      ),
      // TODO: This is horrible split the enums and make LispyError generic over them
      _ => format_string.push_str(""),
  }

  make_error(kind, format!(format_string, self, pair.got, pair.expceted))
}
```

This solution cleaned up the code significantly, but there was still some smell
left. Since you cannot specify which variants of an enum a method accepts, I
had to match on all the errors. But in the `Operate` trait, I only wanted to
handle errors made by the user. This meant that my `ErrorKind` type was not
enough to convey this to the user.

Inorder to differentiate between user error and language error, I split the
`ErrorKind` enum into two different enums `LangError` and `ProgramError` and
changed `ErrorKind` to an empty trait and created the corresponding impls for
the two new enums. Then I made the struct `LispyError` generic with the trait
bound being `ErrorKind + Debug`.

```rs
pub struct LispyError<T>
where
    T: ErrorKind + Debug,
{
    kind: T,
    message: String,
}

pub trait ErrorKind {}

#[derive(Debug)]
pub enum ProgramError {
  // ProgramError variants
}
impl ErrorKind for ProgramError {}

pub enum LangError {
  // LangError variants
}
impl ErrorKind for LangError {}
```

This meant that LispyError's size would be unknown at compile time, therefore,
I had to put `LispyError` in a `Box` and pass that around in my program. Once I
made the required changes, my `handle_error` method was now accepting only
`ProgramError` as it's kind.

```rs
 fn handle_error<G, X>(&self, kind: ProgramError, got: G, expected: X) -> Box<Error>
    where
        Self: Debug,
        G: Debug,
        X: Debug,
```

While the `make_error` method from the `error` module now was generic over T
with the trait bound `ErrorKind`. This meant that I could have the appropriate
error type be enforced by the compiler instead of it being a part of some
comment or documented in a different part of the code base.

```rs
 pub fn make_error<T: ErrorKind + Debug + 'static>(kind: T, message: String) -> Box<Error> {
```

All in all, this exercise taught me about approaching a problem by taking it
one step at a time and incrementally improving an existing situation. At every
step, I built on top of the changed made in the previous step, and the compiler
helping me along the way to ensure that the code I was writing was safe. Plus,
it was a great way to really understand what makes rust so amazingly
productive.  The compiler actually inspired me to improve the error messages in
my program and now, anyone reading the code will be able to clearly understand
the logic behind why `handle_error` only takes a certain type, and the compiler
will ensure that those rules are being obeyed.

The next steps in error handling, if I need something more complex would be
using the `Failure` crate which provides many more features for error handling
in applications with more complex requirements. But for now, and for my usecase
I feel like this system works. Please do let me know if I can improve upon this
in any way or have made any mistakes.
