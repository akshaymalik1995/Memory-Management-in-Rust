# Part 2: Borrowing and References

## Learning Objectives

*   Understand the concept of borrowing in Rust.
*   Learn the rules for immutable and mutable borrows.
*   Recognize how Rust prevents dangling references at compile time.
*   Use references to access data without taking ownership.

---

In Part 1, we learned about Rust's ownership system, where each value has a unique owner, and the value is dropped when the owner goes out of scope. While this ensures memory safety, always transferring ownership can be cumbersome. Imagine having to return ownership of every piece of data a function merely reads! This is where **borrowing** comes in.

## What is Borrowing?

Borrowing allows you to create **references** to a value without taking ownership of it. Think of it like lending a book. You still own the book, but someone else can read it (immutable borrow) or, with special permission, write notes in it (mutable borrow). The key is that you, the owner, expect to get the book back.

A reference is like a pointer in that it’s an address we can follow to access data stored at that address; that data is owned by some other variable. Unlike a pointer, a reference is guaranteed to point to a valid value of a particular type for the life of that reference.

We create references using the `&` symbol.

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1); // Pass a reference to s1

    println!("The length of '{}' is {}.", s1, len);
    // s1 is still valid here because we only borrowed it!
}

fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // Here, s goes out of scope. But because it does not have ownership
  // of what it refers to, nothing happens.
```

In this example:
*   `s1` owns the `String`.
*   We pass `&s1` to `calculate_length`. This creates a reference that *borrows* the value of `s1` but does not own it.
*   The `calculate_length` function takes `s: &String`. The `&` indicates that `s` is a reference.
*   Because `calculate_length` only borrows `s1`, `s1` is still valid and usable after the function call.

This is much more convenient than transferring ownership and then having `calculate_length` return both the length and the `String` back.

## The Borrowing Rules

Rust enforces strict rules about borrowing to maintain memory safety. These rules are checked at compile time:

1.  **One or more immutable references (`&T`) OR exactly one mutable reference (`&mut T`).**
    *   You can have many immutable references to a piece of data because they don’t change the data. Think of many people reading the same document simultaneously.
    *   However, you can only have one mutable reference at a time. This prevents data races, where multiple pointers access the same data simultaneously, and at least one of them is being used to write, leading to inconsistent data.

2.  **References must always be valid.**
    *   Rust ensures that references will never be *dangling* – pointing to memory that has been deallocated or is no longer valid. We'll see how this works shortly.

### Immutable Borrows (`&T`)

Immutable references allow you to read data but not change it.

```rust
fn main() {
    let s = String::from("world");

    let r1 = &s; // immutable borrow
    let r2 = &s; // another immutable borrow - this is fine!

    println!("r1: {}, r2: {}", r1, r2);
    // s is still the owner and valid
}
```

### Mutable Borrows (`&mut T`)

Mutable references allow you to change the data they refer to.

```rust
fn main() {
    let mut s = String::from("hello"); // s must be mutable to allow mutable borrows

    let r1 = &mut s;
    // let r2 = &mut s; // This would be a COMPILE ERROR! Cannot have a second mutable borrow.
    // let r3 = &s;    // This would ALSO be a COMPILE ERROR! Cannot have an immutable borrow when a mutable one exists.

    r1.push_str(", world!");
    println!("{}", r1); // or println!("{}", s); after r1 is no longer used
}
```

Key points for mutable borrows:
*   The original variable (`s` in this case) must be declared with `mut`.
*   You can only have **one** mutable reference to a particular piece of data in a particular scope.

This restriction is powerful because it allows Rust to prevent data races at compile time. A data race has three behaviors:
*   Two or more pointers access the same data at the same time.
*   At least one of the pointers is being used to write to the data.
*   There’s no mechanism being used to synchronize access to the data.

Rust prevents this by not even compiling code with data races!

**Scope of References:**
A reference’s scope starts from where it is introduced and continues through the last time that reference is used. For example:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // immutable borrow
    let r2 = &s; // immutable borrow
    println!("{} and {}", r1, r2);
    // r1 and r2 are no longer used after this point

    let r3 = &mut s; // mutable borrow - THIS IS OKAY!
    r3.push_str(" world");
    println!("{}", r3);
}
```
This code is valid because the last usage of the immutable references `r1` and `r2` is in the `println!`. After that, their scope ends, allowing a mutable reference `r3` to be created.

## Dangling References and How Rust Prevents Them

A dangling reference is a pointer that references a location in memory that may have been given to someone else, by freeing some memory while preserving a pointer to that memory.

In Rust, the compiler guarantees that references will never be dangling references. If you have a reference to some data, the compiler will ensure that the data will not go out of scope before the reference to the data does.

Consider this example, which would lead to a dangling reference in other languages:

```rust
// fn main() {
//     let reference_to_nothing = dangle();
// }

// fn dangle() -> &String { // dangle returns a reference to a String
//     let s = String::from("hello"); // s is a new String

//     &s // we return a reference to the String, s
// } // Here, s goes out of scope, and is dropped. Its memory goes away.
  // Danger!
```

If you try to compile this code, Rust will give you an error:

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the 'static lifetime
  |
5 | fn dangle() -> &'static String {
  |                ~~~~~~~
```

The error message mentions "missing lifetime specifier." We'll cover lifetimes in detail in the next part. For now, the key takeaway is that Rust understands that `s` is created inside `dangle`, and its memory will be deallocated when `dangle` finishes. Returning a reference to `s` would mean the reference points to invalid memory.

**The solution here is to return the `String` directly, transferring ownership:**

```rust
fn main() {
    let s = no_dangle();
    println!("{}", s);
}

fn no_dangle() -> String {
    let s = String::from("hello");
    s // ownership is moved out
}
```
Because ownership is moved out, Rust doesn't need to deallocate anything when `no_dangle` finishes, and the `String` remains valid.

## Common Pitfalls and Compiler Insights

*   **Attempting to have multiple mutable references or a mix of mutable and immutable references to the same data simultaneously.**
    *   **Pitfall:**
        ```rust
        // let mut s = String::from("text");
        // let r1 = &mut s;
        // let r2 = &mut s; // Error!
        // println!("{}, {}", r1, r2);
        ```
        Or:
        ```rust
        // let mut s = String::from("text");
        // let r1 = &s;
        // let r2 = &mut s; // Error!
        // println!("{}, {}", r1, r2);
        ```
    *   **Compiler Insight:** Rust will error with messages like `cannot borrow s as mutable more than once at a time` or `cannot borrow s as mutable because it is also borrowed as immutable`.
    *   **Fix:** Ensure that mutable borrows are exclusive. If you need multiple references for reading, make them all immutable. If you need to mutate, ensure no other references (mutable or immutable) exist at that time.

*   **Modifying data through an immutable reference.**
    *   **Pitfall:**
        ```rust
        // let s = String::from("text");
        // let r1 = &s;
        // r1.push_str(" more"); // Error!
        ```
    *   **Compiler Insight:** The compiler will state `cannot borrow *r1 as mutable, as it is behind a & reference`.
    *   **Fix:** If you need to modify the data, you must use a mutable reference (`&mut T`), and the original variable must also be mutable (`let mut s`).

*   **Creating a reference to a value that goes out of scope before the reference does (leading to a dangling reference).**
    *   **Pitfall:** As seen in the `dangle()` example.
    *   **Compiler Insight:** Rust will typically error with a message about lifetimes, preventing the compilation of code that would create a dangling reference.
    *   **Fix:** Ensure the data outlives any references to it. This often means returning ownership or restructuring code, which lifetimes (covered next) help manage explicitly.

## Exercises/Challenges

1.  **Identify the Error:** The following code has an error. Explain what it is and why it occurs. How would you fix it?
    ```rust
    fn main() {
        let mut list = vec![1, 2, 3];
        let first = &list[0];

        list.push(4); // Problematic line

        // println!("The first element is: {}", first);
    }
    ```
    *(Hint: Consider what `push` might do to the vector's memory and how it relates to existing references.)*

2.  **Mutable Borrow Scope:** Explain why the following code compiles successfully, even though it seems like we have an immutable borrow (`r1`) and then a mutable borrow (`r2`) of `s`.
    ```rust
    fn main() {
        let mut s = String::from("foo");
        let r1 = &s;
        println!("r1: {}", r1);
        // r1 is no longer used after this point

        let r2 = &mut s;
        r2.push_str("bar");
        println!("r2: {}", r2);
    }
    ```

3.  **Write a Function:** Write a function `append_greeting` that takes a mutable reference to a `String` and appends ", world!" to it. Show how to call this function.

## Key Takeaways/Summary

*   Borrowing allows access to data using references (`&T` or `&mut T`) without taking ownership.
*   You can have many immutable references (`&T`) OR one mutable reference (`&mut T`), but not both at the same time for the same data in the same scope.
*   References must always be valid; Rust's compiler prevents dangling references.
*   Using references is crucial for writing efficient Rust code that avoids unnecessary data copying or complex ownership transfers.

---

Understanding ownership and borrowing is fundamental to Rust. However, there are situations, especially with references in more complex data structures or function signatures, where we need to be more explicit about *how long* references are valid. This brings us to the concept of **lifetimes**, which we will explore in Part 3.
