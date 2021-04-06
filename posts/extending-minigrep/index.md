# A Beginner's Guide to Handling Errors in Rust

_April 5th, 2021 · #rust · #error_handling · #beginner_

The example projects in _The Rust Programming Language_ are great for introducing new would-be Rustaceans to different aspects and features of Rust. In this post, we'll be looking at some different ways of implementing a more robust error handling infrastructure but bolstering the `minigrep` project from _The Rust Programming Language_.

The `minigrep` project is introduced in [chapter 12][ch12] and walks the reader through building a simple version of the `grep` command line tool, which is a utility for searching through text. For example, you'd pass in a query, the text you're searching for, along with the file name where the text lives, and get back all of the lines that contain the query text. 

The goal of this post is to extend the book's `minigrep` implementation with more robust error handling patterns so that you'll have a better idea of different ways to handle errors in a Rust project.

For reference, you can find the final code for the book's version of `minigrep` [here][minigrep-orig].

## Error handling use cases

A common pattern when it comes to structuring Rust projects is to have a "library" portion where the primary data structures, functions, and logic live and an "application" portion that ties the library functions together. 

You can see this in the file structure of the original `minigrep` code: the application logic lives inside of the `src/bin/main.rs` file, and it's merely a thin wrapper around data structures and functions that are defined in the `src/lib.rs` file; all the `main` function does is call `minigrep::run`. 

This is important to point out because depending on whether we're building an application or a library changes how we approach error handling. 

When it comes to an application, the end user most likely doesn't want to know about the nitty gritty details of what caused an error. Indeed, the end user of an application probably only ought to be notified of an error in the event that the error is _unrecoverable_. In this case, it's also useful to provide details on why the unrecoverable error occurred, especially if it has to do with user input. If some sort of recoverable error happened in the background, the consumer of an application probably doesn't need to know about it.

Conversely, when it comes to a library, the end users are other developers who are using the library and building something on top of it. In this case, we'd like to give as many relevant details about any errors that occurred in our library as possible. The consumer of the library will then decide how they want to handle those errors. 

So how do these two approaches play together when we have both a library portion and an application portion in our project? The `main` function executes the `minigrep::run` function and outputs any errors that crop up as a result. So most of our error handling efforts will be focused on the library portion.  

## Surfacing library errors

In `src/lib.rs`, we have two functions, `Config::new` and `run`, which might return errors:
```rust=
impl Config {
        pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
            args.next();

        let query = match args.next() {
                Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
                Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config {
                query,
            filename,
            case_sensitive,
        })
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
        let contents = fs::read_to_string(config.filename)?;

    let results = if config.case_sensitive {
            search(&config.query, &contents)
    } else {
            search_case_insensitive(&config.query, &contents)
    };

    for line in results {
            println!("{}", line);
    }

    Ok(())
}
```

There are exactly three spots where errors are being returned: two errors occur in the `Config::new` function, which returns a `Result<Config, &'static str>`. In this case, the error variant of the `Result` is a static string slice.

Here we return an error when a query is not provided by the user.
```rust=
let query = match args.next() {
        Some(arg) => arg,
    None => return Err("Didn't get a query string"),
};
```

Here we return an error when a filename is not provided by the user.
```rust=
let filename = match args.next() {
        Some(arg) => arg,
    None => return Err("Didn't get a file name"),
};
```

The main problem with structuring our errors in this way as static strings is that the error messages are not located in a central spot where we can easily refactor them should we need to. It also makes it more difficult to keep our error messages consistent between the same types of errors.

The third error occurs at the top of `run` function, which returns a `Result<(), Box<dyn Error>>`. The error variant in this case is a [trait object][trait-object] that implements the `Error` [trait][error-trait]. In other words, the error variant for this function is any instance of a type that implements the `Error` trait. 

Here we bubble up any errors that might have occurred as a result of calling `fs::read_to_string`.
```rust=
let contents = fs::read_to_string(config.filename)?;
```

This works for the errors that might crop up as a result of calling `fs::read_to_string` since this function is capable of returning multiple types of errors. Therefore, we need a way to generically represent those different possible error types; the commonality between them all is the fact that they all implement the `Error` trait! 

Ultimately, what we want to do is define all of these different types of errors in a central location and have them all be variants of a single type. 

### Defining error variants in a central type

We'll create a new `src/error.rs` file and define an enum `AppError`, deriving the `Debug` trait in the process so that we can get a debug representation should we need it. We'll name each of the variants of this enum in such a way that they appropriately represent each of the three types of errors:

```rust=
#[derive(Debug)]
pub enum AppError {
        MissingQuery,
    MissingFilename,
    ConfigLoad,
}
```

The third variant, `ConfigLoad`, maps to the error that might crop up when calling `fs::read_to_string` in the `Config::run` function. This might seem a bit out of place at first, since if an error occurs with that function, it would be some sort of I/O problem reading the provided config file. So why didn't we name it `IOError` or something like that? 

In this case, since we're surfacing an error from a standard library function, it's more relevant to our application to describe how the surfaced error affects it, instead of simply reiterating it. When an error occurs with `fs::read_to_string`, that prevents our `Config` from loading, so that's why we named it `ConfigLoad`. 

Now that we have this type, we need to update all of the spots in our code where we return errors to utilize this `AppError` enum.

### Returning variants of our `AppError`

At the top of our `src/lib.rs` file, we need to declare our `error` module and bring `error::AppError` into scope:
```rust=
mod error;

use error::AppError;
``` 

In our `Config::new` function, we need to update the spots where we were returning static string slices as errors, as well as the return type of the function itself:

```diff=
- pub fn new(mut args: env::Args) -> Result<Config, &'static str>
+ pub fn new(mut args: env::Args) -> Result<Config, AppError>
    // --snip--
    
    let query = match args.next() {
            Some(arg) => arg,
-       None => return Err("Didn't get a query string"),
+       None => return Err(AppError::MissingQuery),
    };

    let filename = match args.next() {
            Some(arg) => arg,
-       None => return Err("Didn't get a file name"),
+       None => return Err(AppError::MissingFilename),
    };
    
    // --snip--
```

The third error in the `run` function only requires us to update its return type, since the `?` operator is already taking care of bubbling the error up and returning it should it occur.

```diff=
- pub fn run(config: Config) -> Result<(), Box<dyn Error>>
+ pub fn run(config: Config) -> Result<(), AppError>
```

Ok, so we're now making use of our error variants, which, should they occur, are being surfaced to our `main` function and printed out. But we no longer have the actual error messages that we had before defined anywhere! 

### Annotating error variants with `thiserror`

The `thiserror` [crate][thiserror] is one that is commonly used to provide an ergonomic way to format error messages in a Rust library. 

It allows us to annotate each of the variants in our `AppError` enum with the actual error message that we want displayed to the end user. 

Let's add it as a dependency in our Cargo.toml:

```
[dependencies]
thiserror = "1"
```

In `src/error.rs` we'll bring the `thiserror::Error` trait into scope and have our `AppError` type derive it. We need this trait derived in order to annotate each enum variant with an `#[error]` block. Now we specify the error message that we want displayed for each particular variant: 

```diff=
+ use std::io;

+ use thiserror::Error;

- #[derive(Debug)]
+ #[derive(Debug, Error)]
pub enum AppError {
    +   #[error("Didn't get a query string")]
    MissingQuery,
+   #[error("Didn't get a file name")]
    MissingFilename,
+   #[error("Could not load config")]
    ConfigLoad {
    +       #[from] 
+       source: io::Error,
+   }
}
```

What's all the extra stuff was added to the `ConfigLoad` variant? Since a `ConfigLoad` error only occurs when there's an underlying error with the call to `fs::read_to_string`, what the `ConfigLoad` variant is actually doing is providing extra context around the underlying I/O error. 

`thiserror` allows us to wrap a lower-level error in additional context by annotating it with a `#[from]` in order to convert the `source` into our homebrew error type. In this way, when an I/O error occurs (like when we specify a file to search through that doesn't actually exist), we get an error like this:

```
Could not load config: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

Without it, the resulting error message looks like this:

```
Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

To a consumer of our library, it's harder to figure out the source of this error; the additional context helps a lot. 

You can find the version of `minigrep` that uses `thiserror` [here][minigrep-thiserror].

## A more manual approach

Now we'll switch gears and look out how we might achieve the same results that `thiserror` provides us, but without bringing it in as a dependency. 

Under the hood, `thiserror` performs some magic with procedural macros, which can have a noticeable effect on compilation speeds. In the case of `minigrep`, we have very few error variants and the project is so small that a dependency on `thiserror` really won't introduce much of an increase in compilation time, but it could be a consideration in a much larger and more complex project. 

So on that note, we'll wrap up this post by ripping it out and replacing it with our own hand-rolled implementation. The nice thing about going down this route is that we'll only need to make changes to the `src/error.rs` file to implement all of the necessary changes (besides, of course, removing `thiserror` from our Cargo.toml).

```diff=
[dependencies]
- thiserror = "1"
```

Let's remove all of the annotations that `thiserror` was providing us. We'll also replace the `thiserror::Error` trait with the `std::error::Error` trait:

```diff=
- use thiserror::Error;
+ use std::error::Error;

- #[derive(Debug, Error)]
+ #[derive(Error)]
pub enum AppError {
    -   #[error("Didn't get a query string")]
    MissingQuery,
-   #[error("Didn't get a file name")]
    MissingFilename,
-   #[error("Could not load config")]
    ConfigLoad {
    -      #[from]
       source: io::Error,
    }
}
```

In order to get back all of the functionality we just wiped, we'll need to do three things:

1. Implement the `Display` [trait][display-docs] for `AppError` so that our error variants can be displayed to the user.
2. Implement the `Error` [trait][error-docs] for `AppError`. This trait represents the basic expectations of an error type, namely that they implement `Display` and `Debug`, plus the capability to fetch the underlying source or cause of the error. 
3. Implement `From<io::Error>` for `AppError`. This is required so that we can convert an I/O error returned from `fs::read_to_string` into an instance of `AppError`. 

Here's our implementation of the `Display` trait for our `AppError`. It maps each error variant to an string and writes it to the `Display` formatter.

```rust=
use std::fmt;

impl fmt::Display for AppError {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            match self {
                Self::MissingQuery => f.write_str("Didn't get a query string"),
            Self::MissingFilename => f.write_str("Didn't get a file name"),
            Self::ConfigLoad { source } => write!(f, "Could not load config: {}", source),
        }
    }
}
```

And here's our implementation of the `Error` trait. The main method to be implemented is the `Error::source` method, which is meant to provide information about the source of an error. For our `AppError` type, only `ConfigLoad` exposes any underlying source information, namely the I/O error that might happen as a result of calling `fs::read_to_string`. There's no underlying source information to expose in the case of the other error variants.

```rust=
use std::error;

impl error::Error for AppError {
        fn source(&self) -> Option<&(dyn Error + 'static)> {
            match self {
                Self::ConfigLoad { source } => Some(source),
            _ => None,
        }
    }
}
```

The `&(dyn Error + 'static)` part of the return type is similar to the `Box<dyn Error>` trait object that we saw earlier. The main difference here is that the trait object is behind an immutable reference instead of a `Box` pointer. The `'static` lifetime here means the trait object itself only contains owned values, i.e., it doesn't store any references internally. This is necessary in order to assuage the compiler that there's no chance of a dangling pointer here.

Lastly, we need a way to convert an `io::Error` into an `AppError`. We'll do this by impling `From<io::error> for AppError`.

```rust=
impl From<io::Error> for AppError {
        fn from(source: io::Error) -> Self {
            Self::ConfigLoad { source }
    }
}
```

There's not much to this one. If we get an `io::Error`, all we do to convert it to an `AppError` is wrap it in a `ConfigLoad` variant. 

And that's all folks! You can find this version of our minigrep implementation [here][minigrep-manual].

## Summary

In closing, we discussed how the original `minigrep` implementation presented in *The Rust Programming Language* book is a bit lacking in the error handling department, as well as how to think about different error handling use cases. 

From there, we showcased how to use the `thiserror` crate to centralize all of the possible error variants into a single type. 

Finally, we peeled back the veneer that `thiserror` provides and showed how to replicate the same functionality manually.

I hope you all learned something from this post! 🙂


[ch12]: https://doc.rust-lang.org/book/ch12-00-an-io-project.html
[minigrep-orig]: https://github.com/seanchen1991/error-handling-examples/tree/minigrep-control/examples/minigrep
[minigrep-thiserror]: https://github.com/seanchen1991/error-handling-examples/tree/minigrep-thiserror/examples/minigrep
[minigrep-manual]: https://github.com/seanchen1991/error-handling-examples/tree/main/examples/minigrep
[trait-object]: https://doc.rust-lang.org/book/ch17-02-trait-objects.html#defining-a-trait-for-common-behavior
[error-trait]: https://doc.rust-lang.org/std/error/trait.Error.html
[thiserror]: https://docs.rs/thiserror/1.0.24/thiserror/
[display-docs]: https://doc.rust-lang.org/std/fmt/trait.Display.html
[error-docs]: https://doc.rust-lang.org/std/error/trait.Error.html
