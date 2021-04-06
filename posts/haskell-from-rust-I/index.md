# Haskell::From(Rust) I: Infix Notation and Currying
_July 20th, 2020 | #haskell | #rust | #exercism_

## Prelude

I’ve been meaning to learn Haskell for a while now. Most people who write Rust likely have had at least a background-level exposure to Haskell (if they hadn’t already encountered/learned the language on their own prior). Since the language (along with ML and possibly OCaml?) had such an effect on Rust’s development, digging into it will likely pay dividends in the form of improving my understanding of Rust. 

This _Haskell::From(Rust)_ series will chronicle some of the learnings I glean from learning Haskell, as well as the takeaways that can be applied to write better code in Rust. 

## The Setup

While working through Exercism’s Haskell track, specifically the problem asking you to implement a [pangram](https://en.wikipedia.org/wiki/Pangram) checker, I encountered a confusion point regarding [infix functions](https://wuciawe.github.io/functional%20programming/haskell/2016/07/03/infix-functions-in-haskell.html). From what I can gather, this is not an uncommon sticking point for those who are new to Haskell.

In order to solve this pangram problem, I opted to sort the input string and filter out any duplicates: 

```haskell
isPangram text = [‘a’..’z’] == (List.nub . List.filter (/=‘ ‘) . List.sort $ text)
```

Afterwards, I browsed some others’ implementations and came across this one:

```haskell
isPangram text = all (`elem` lowercased) [‘a’..’z’]
	where lowercased = map toLower text
```

## The Turn

I read ``(`elem` lowercased) [‘a’..’z’]`` as checking that the first element in the `lowercased` string is a member of the set `[‘a’..’z’]`. The `all` function then facilitates iterating through all of the characters of `lowercased`, ensuring that no iterations return false. 

In short, I thought this line checked that every character in the input string was a lower-cased letter. But that shouldn’t be sufficient for implementing a pangram function. Upon testing out the code, it did indeed work (passing it an input string that didn’t contain all of the English letters caused it to return false), so clearly there was a hole in my understanding of this logic. 

## The Prestige

After doing some research and posting my question on Haskell forums, I found an explanation that clicked:

> Any operator (including `foo`) can be written in prefix notation by surrounding it with parentheses: `(+)`, `(==)`, ``(`div`)``. These are called _sections_, and they’re regular functions like `map` and `sin`. For example, instead of `2 + 2`, you can write `(+) 2 2`. You can even leave off either the left or right operand to curry an operator. For example, ``(`div` 0)`` is a function that takes one argument and divides it by zero.

- _Haskell: The Confusing Parts_ [1](http://echo.rsmw.net/n00bfaq.html)

The last sentence is what made it click for me: this was an example of [currying](https://en.wikipedia.org/wiki/Currying). Essentially, ``(`elem` lowercased)`` leveraged currying to define a new function that takes one argument and checks that that argument is an element of `lowercased`. My understanding had been flipped based off of how I read the function, which was heavily influenced by what programming languages I’d had prior experience with. 

Clearly, if one comes from an imperative or OOP background, this sort of syntax can be strange. Even Rust, with its relatively strong emphasis on functional norms (especially when it comes to chaining iterator adapters), isn’t nearly as expressive when it comes to the number of ways in which functions can be defined in comparison to Haskell. 

This is, however, a feature of Haskell that I find quite enticing. I’m looking forward to learning more and digging in deeper! 

## Before We Part Ways

I couldn’t leave my dear readers without at least a bit of Rust code. That’d be inexcusable! 

Using the same pieces present in the Haskell implementation, we can implement a pangram function in Rust like this:

```rust
fn is_pangram(text: &str) -> bool {
	// maps to `[‘a’..’z’]`
    let alphabet = “abcdefghijklmnopqrstuvwxyz”;
    
    // maps to `where lowercased = map toLower text`
    let lowercased = text.to_lowercase();
    
    // maps to `all (`elem` lowercased)`
    alphabet.chars().all(|letter| lowercased.contains(letter))
}
```

Not as elegant as the Haskell version, but also not too far off in my opinion. 

**_Edit_**

A reader, Maxwell Orok, reached out to me to let me know that char ranges are a thing in Rust now as of version [1.45](https://blog.rust-lang.org/2020/07/16/Rust-1.45.0.html). This is great, as it allows us to clean up the Rust implementation a bit:

```rust
fn is_pangram(text: &str) -> bool {
    // maps to `[‘a’..’z’]`
    // We can now collect into a `String` from a `Range`
    // instead of having to hard code the string
    let alphabet: String = ('a'..='z').collect();
    
    // maps to `where lowercased = map toLower text`
    let lowercased = text.to_lowercase();
    
    // maps to `all (`elem` lowercased)`
    alphabet.chars().all(|letter| lowercased.contains(letter))
}
```

Definitely much better in my opinion!
