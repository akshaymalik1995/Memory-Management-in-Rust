# Part 1: Introduction to Ownership

## Learning Objectives

*   Understand the concept of ownership in Rust.
*   Learn the three core rules of ownership.
*   Differentiate between moving, copying, and cloning data.
*   Grasp how scope affects ownership and data lifetimes.

---

Welcome to the first part of our series on mastering memory management in Rust! Rust's approach to memory management is one of its most distinctive and powerful features, enabling it to guarantee memory safety and prevent common bugs like dangling pointers or data races without needing a garbage collector. At the heart of this system is the concept of **ownership**.

## What is Ownership?

In many programming languages, you have to manually manage memory (like in C/C++) or rely on a garbage collector (like in Java or Python). Rust takes a different path with ownership.

Think of ownership like this: imagine a valuable item. Only one person can truly "own" it at any given time. If they give it to someone else, they no longer own it. If they want to share it temporarily, they can "lend" it. Rust applies similar rules to how programs manage memory.

The core idea is that each value in Rust has a variable that’s its *owner*. When the owner goes out of scope, the value will be dropped (i.e., its memory will be deallocated). This system provides control over memory lifetime and ensures memory safety at compile time.

## The Three Ownership Rules

Rust’s ownership system is governed by three fundamental rules:

1.  **Each value in Rust has an owner.**
2.  **There can only be one owner at a time.**
3.  **When the owner goes out of scope, the value will be dropped.**

Let's explore these rules with examples.

### Rule 1 & 3: Owner and Scope

```rust
fn main() {
    // s is not valid here, it’s not yet declared
    {
        let s = String::from("hello"); // s is valid from this point forward

        // do stuff with s
        println!("{}", s);
    } // this scope is now over, and s is no longer valid
      // Rust calls `drop` automatically here for `s`
}
```

In this example:
*   When `s` comes into scope (it's declared), it becomes the owner of the `String` data "hello". This `String` is allocated on the heap.
*   While `s` is valid, we can use it.
*   When `s` goes out of scope (the inner block `{}` ends), Rust automatically calls a special function called `drop` for the `String`. This function deallocates the memory that was holding "hello".

This automatic cleanup is a key part of Rust's safety.

### Rule 2: Only One Owner at a Time - Move

What happens when we try to assign a value owned by one variable to another?

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // s1's ownership is moved to s2

    // println!("s1 is: {}", s1); // This would cause a compile-time error!
    println!("s2 is: {}", s2);
}
```

Here, `s1` initially owns the `String` data. When we write `let s2 = s1;`, Rust doesn't just copy the pointer to the data. Instead, it **moves** ownership from `s1` to `s2`. After the move:
*   `s2` is now the owner.
*   `s1` is no longer valid and cannot be used.

If you try to use `s1` after the move, the Rust compiler will stop you with an error: `borrow of moved value: s1`. This prevents a "double free" error, where both `s1` and `s2` might try to free the same memory when they go out of scope.

This "move" behavior is default for types that manage resources on the heap, like `String`, `Vec<T>`, or `Box<T>`.

## Moving, Copying, and Cloning

### Move (Default for Heap Data)

As we saw with `String`, types that store data on the heap are moved by default. This is efficient because it avoids copying potentially large amounts of data.

### Copy (for Stack Data)

What about simple types like integers, booleans, or characters?

```rust
fn main() {
    let x = 5;
    let y = x; // x's value is copied to y

    println!("x = {}, y = {}", x, y); // Both x and y are valid
}
```

This code works perfectly fine. Why? Because types like integers have a known size at compile time and are stored entirely on the stack. Copying them is cheap and fast. Rust has a special annotation called the `Copy` trait. If a type implements the `Copy` trait, an older variable is still usable after assignment.

Common `Copy` types include:
*   All integer types (e.g., `i32`, `u64`).
*   The boolean type (`bool`).
*   All floating-point types (e.g., `f32`, `f64`).
*   The character type (`char`).
*   Tuples, if they only contain types that are also `Copy`. For example, `(i32, bool)` is `Copy`, but `(i32, String)` is not.

If a type, or any of its parts, has implemented the `Drop` trait (which `String` does for deallocation), then it cannot implement `Copy`.

### Clone (Explicit Deep Copy)

What if we want to deeply copy heap data, like a `String`, so that we have two distinct instances? We use the `clone()` method.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone(); // s2 is a deep copy of s1

    println!("s1 = {}, s2 = {}", s1, s2); // Both are valid and own their data
}
```
When you call `clone()`, Rust allocates new memory on the heap and copies the content of `s1` into it. Both `s1` and `s2` are now valid and own their respective `String` data. Cloning can be an expensive operation for large data structures.

## Ownership and Functions

Passing a value to a function works similarly to assigning it to a variable.

```rust
fn main() {
    let s = String::from("world"); // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here

    // println!("{}", s); // This would be a compile-time error!

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it’s okay to still
                                    // use x afterward
    println!("x is still: {}", x);
} // Here, x goes out of scope, then s. But because s's value was moved, nothing
  // special happens for s.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

**Returning values also transfers ownership:**

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3
    // println!("{}", s2); // s2 is moved, no longer valid!
    println!("s1: {}, s3: {}", s1, s3);
} // Here, s3 goes out of scope and is dropped. s2 was moved, so nothing
  // happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it
    let some_string = String::from("yours"); // some_string comes into scope
    some_string                              // some_string is returned and
                                             // moves out to the calling function
}

// This function takes a String and returns one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope
    a_string  // a_string is returned and moves out to the calling function
}
```

The mechanics of passing values to functions and returning values from them are consistent with the ownership rules: ownership is always moved unless the type implements the `Copy` trait.

## Scope and Lifetimes (Brief Introduction)

We've seen that values are dropped when their owner goes out of scope. The *scope* of a variable is the region of a program where it is valid.

The term *lifetime* refers to a scope for which a reference is valid. We'll dive much deeper into lifetimes in Part 3, especially in the context of borrowing, but it's important to understand that ownership and scopes are intrinsically linked to how long data lives in your program.

## Common Pitfalls and Compiler Insights

*   **Using a variable after it has been moved:**
    *   **Pitfall:** `let s1 = String::from("text"); let s2 = s1; println!("{}", s1);`
    *   **Compiler Insight:** The compiler will issue a "borrow of moved value" error. This is Rust preventing a potential use-after-free or double-free bug.
    *   **Fix:** If you need both variables to be valid and independent, use `s2 = s1.clone();`. If you only need one, the move is correct.

*   **Assuming complex types are `Copy`:**
    *   **Pitfall:** Expecting types like `Vec<T>` or `String` to behave like `i32` during assignment (i.e., be copied implicitly).
    *   **Compiler Insight:** If you try to use the original variable after assigning it to another (a move), you'll get the "borrow of moved value" error.
    *   **Fix:** Understand which types are `Copy` and which are `Move`. Use `clone()` for explicit deep copies when needed.

## Exercises/Challenges

1.  **Explain the Output:** What happens in the following code, and why? Will it compile? If not, how would you fix it while ensuring both `v1` and `v2` can be used?
    ```rust
    fn main() {
        let v1 = vec![1, 2, 3];
        let v2 = v1;
        println!("v1[0] is: {}", v1[0]);
    }
    ```

2.  **Ownership Transfer:** Write a function `process_data` that takes ownership of a `String`, prints its length, and then returns ownership of the same `String` back to the caller. Show how you would call this function.

3.  **Copy vs. Clone:** Create a struct `Point { x: i32, y: i32 }`.
    *   Make it so that `Point` instances are copied when assigned, not moved.
    *   Then, create another struct `UserData { name: String, age: u32 }`. Explain why `UserData` cannot be made `Copy` in the same way as `Point` without further changes. How would you create an independent copy of a `UserData` instance?

## Key Takeaways/Summary

*   Ownership is Rust's mechanism for memory safety without a garbage collector.
*   Three rules: one owner, owner per value, value dropped when owner out of scope.
*   Data on the heap (like `String`, `Vec<T>`) is **moved** by default.
*   Data on the stack (like `i32`, `bool` and types implementing `Copy` trait) is **copied**.
*   The `clone()` method provides a way to perform a deep copy of heap data.
*   Function calls and returns also transfer ownership.

---

In the next part, we'll explore **borrowing and references**, which allow us to access data without taking ownership. This is crucial for writing flexible and efficient Rust code.
