# Part 6: Interior Mutability

## Learning Objectives

*   Understand the concept of interior mutability and why it's needed.
*   Learn to use `Cell<T>` for mutating `Copy` types when they are otherwise immutable.
*   Explore `RefCell<T>` for mutating data when you have an immutable reference, with runtime borrow checking.
*   Recognize when and why to use `Cell<T>` vs. `RefCell<T>`.
*   Understand the borrowing rules enforced by `RefCell<T>` at runtime.

---

In Rust, the borrowing rules are usually enforced at compile time: you can have multiple immutable references (`&T`) or one mutable reference (`&mut T`), but not both. This is a cornerstone of Rust's safety guarantees. However, sometimes these rules can be too restrictive, especially in scenarios where a type needs to mutate itself internally even when it appears immutable from the outside.

This is where **interior mutability** comes in. It's a design pattern in Rust that allows you to mutate data even when there are immutable references to that data. Rust provides several types in its standard library that enable interior mutability, primarily `Cell<T>` and `RefCell<T>`.

Crucially, interior mutability doesn't throw Rust's safety rules out the window. Instead, it moves the enforcement of borrowing rules from compile time to runtime for specific types. If you violate the rules with these types, your program will panic at runtime instead of failing to compile.

## Why Interior Mutability?

Consider a scenario where you have a struct, and you want to allow some internal state to change, but the methods that change it only have `&self` (an immutable reference to the struct). Without interior mutability, this wouldn't be possible.

Interior mutability is also essential when working with shared-ownership smart pointers like `Rc<T>`. As we saw in Part 5, `Rc<T>` only gives you shared (immutable) access to the underlying data. If you need to mutate data inside an `Rc<T>`, you'll need an interior mutability type.

## `Cell<T>`: For `Copy` Types

`Cell<T>` provides interior mutability for types `T` that implement the `Copy` trait. It allows you to get and set the value inside it, even through an immutable reference to the `Cell<T>` itself.

`Cell<T>` works by simply copying bits in and out. It doesn't use references or borrowing in the typical Rust sense for its `get` and `set` operations, which is why it's restricted to `Copy` types.

```rust
use std::cell::Cell;

struct Point {
    x: i32,
    y: i32,
    cached_sum: Cell<Option<i32>>, // Cache for x + y
}

impl Point {
    fn new(x: i32, y: i32) -> Point {
        Point { x, y, cached_sum: Cell::new(None) }
    }

    fn sum(&self) -> i32 {
        match self.cached_sum.get() {
            Some(sum) => {
                println!("Returning cached sum: {}", sum);
                sum
            }
            None => {
                let sum = self.x + self.y;
                println!("Calculating and caching sum: {}", sum);
                self.cached_sum.set(Some(sum)); // Mutating through &self!
                sum
            }
        }
    }
}

fn main() {
    let p = Point::new(10, 20);
    println!("First call to sum: {}", p.sum());  // Calculates and caches
    println!("Second call to sum: {}", p.sum()); // Uses cached value

    // We can also directly set values if we have a Cell field
    // p.cached_sum.set(Some(100)); // This would be possible if cached_sum were public
}
```

In this example:
*   `Point` has a `cached_sum` field of type `Cell<Option<i32>>`.
*   The `sum` method takes `&self` (an immutable reference).
*   Inside `sum`, we can call `self.cached_sum.get()` to retrieve the current value and `self.cached_sum.set(...)` to store a new value. This mutation happens despite `self` being an immutable reference.
*   `Cell<T>` is suitable here because `Option<i32>` is a `Copy` type (since `i32` is `Copy` and `Option` containing `Copy` types is `Copy`).

**Key characteristics of `Cell<T>`:**
*   Only works with `T` that implements `Copy`.
*   `get()`: Returns a copy of the contained value.
*   `set(value: T)`: Replaces the contained value with `value`.
*   No runtime borrow checking overhead because it just copies bits.
*   Not thread-safe (cannot be `Sync` if `T` is not `Sync`, and `Cell` itself is not `Send` if `T` is not `Send`, generally used in single-threaded contexts).

## `RefCell<T>`: Runtime Borrow Checking

What if the data you want to mutate isn't `Copy`? Or what if you need to get a reference to the data inside, not just copy it out? This is where `RefCell<T>` comes in.

`RefCell<T>` allows you to borrow its contained data mutably (`&mut T`) or immutably (`&T`), even when the `RefCell<T>` itself is behind an immutable reference (e.g., `&RefCell<T>`).

Instead of compile-time borrow checking, `RefCell<T>` enforces Rust's borrowing rules **at runtime**. If you violate these rules (e.g., try to get a mutable borrow while an immutable borrow is active, or try to get multiple mutable borrows), your program will **panic**.

```rust
use std::cell::RefCell;
use std::rc::Rc;

pub trait Messenger {
    fn send(&self, msg: &str);
}

struct MessageQueue {
    messages: RefCell<Vec<String>>,
}

impl MessageQueue {
    fn new() -> MessageQueue {
        MessageQueue { messages: RefCell::new(vec![]) }
    }
}

impl Messenger for MessageQueue {
    fn send(&self, msg: &str) { // Takes &self
        // self.messages.push(String::from(msg)); // This would NOT compile if messages was Vec<String>

        // With RefCell, we can borrow mutably at runtime:
        self.messages.borrow_mut().push(String::from(msg));
        println!("Message sent: {}", msg);
    }
}

fn main() {
    let mq = MessageQueue::new();
    mq.send("Hello from RefCell!");
    mq.send("Another message.");

    // Let's see the messages
    let messages_ref = mq.messages.borrow(); // Immutable borrow
    println!("Current messages: {:?}", messages_ref);

    // Attempting a mutable borrow while an immutable one exists would panic:
    // let mut messages_mut_ref = mq.messages.borrow_mut(); // This would PANIC!
    // messages_mut_ref.push(String::from("Panic attempt"));

    // Dropping the immutable borrow allows a mutable borrow again
    drop(messages_ref);

    let mut messages_mut_ref_ok = mq.messages.borrow_mut();
    messages_mut_ref_ok.push(String::from("Message after drop"));
    println!("Messages after modification: {:?}", messages_mut_ref_ok);
}
```

**Key methods of `RefCell<T>`:**

*   `borrow(&self) -> Ref<T>`: Returns a smart pointer `Ref<T>` for immutable access. Increments an internal immutable borrow counter. Panics if a mutable borrow is active.
*   `borrow_mut(&self) -> RefMut<T>`: Returns a smart pointer `RefMut<T>` for mutable access. Increments an internal mutable borrow counter (which should only go from 0 to 1). Panics if any other borrow (mutable or immutable) is active.
*   `try_borrow(&self) -> Result<Ref<T>, BorrowError>`: Like `borrow`, but returns a `Result` instead of panicking.
*   `try_borrow_mut(&self) -> Result<RefMut<T>, BorrowMutError>`: Like `borrow_mut`, but returns a `Result` instead of panicking.

`Ref<T>` and `RefMut<T>` are smart pointers that implement `Deref` (and `DerefMut` for `RefMut<T>`) to provide access to the inner `T`. When they go out of scope, they automatically decrement the borrow counters in the `RefCell<T>`.

**Combining `Rc<T>` and `RefCell<T>`:**

A very common pattern is `Rc<RefCell<T>>`. This combination allows you to have data with multiple owners (`Rc<T>`) that can also be mutated internally (`RefCell<T>`).

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct SharedData {
    value: i32,
}

fn main() {
    let shared_item = Rc::new(RefCell::new(SharedData { value: 5 }));

    let owner1 = Rc::clone(&shared_item);
    let owner2 = Rc::clone(&shared_item);

    // Owner 1 modifies the data
    owner1.borrow_mut().value += 10;
    println!("Owner1 modified. Current item: {:?}", owner1.borrow());

    // Owner 2 also modifies the data
    {
        let mut borrowed_by_owner2 = owner2.borrow_mut();
        borrowed_by_owner2.value *= 2;
        println!("Owner2 modifying. Current item: {:?}", borrowed_by_owner2);
    } // mutable borrow by owner2 is released here

    println!("Final item value (via shared_item): {:?}", shared_item.borrow());
    println!("Final item value (via owner1): {:?}", owner1.borrow());
    println!("Final item value (via owner2): {:?}", owner2.borrow());
}
```

**Important Notes for `RefCell<T>`:**
*   `RefCell<T>` is also only for use in **single-threaded** scenarios. It is not `Sync`. If you need interior mutability across threads, you would use `Mutex<T>` or `RwLock<T>` (often with `Arc<T>`).
*   The borrow checking is done at runtime. Violations cause a panic.
*   It's useful when you are sure your code adheres to the borrowing rules but the compiler can't verify it (e.g., in some graph structures, or when implementing certain data structures or callback mechanisms).

## `Cell<T>` vs. `RefCell<T>`

| Feature          | `Cell<T>`                                  | `RefCell<T>`                                       |
|------------------|--------------------------------------------|----------------------------------------------------|
| **Data Type**    | `T` must be `Copy`                         | `T` can be any type                                |
| **Access**       | `get()` (copies value), `set()` (replaces) | `borrow()` (`&T`), `borrow_mut()` (`&mut T`)       |
| **Borrow Check** | None (copies bits)                         | Runtime (panics on violation)                      |
| **Overhead**     | Minimal                                    | Some runtime overhead for borrow tracking          |
| **Use Case**     | Simple mutation of `Copy` types            | Mutating non-`Copy` types; getting references      |
| **Thread Safety**| Not `Sync` (typically)                     | Not `Sync`                                         |

Choose `Cell<T>` when you have a `Copy` type and you just need to replace the whole value. Choose `RefCell<T>` when you need to mutate non-`Copy` types or when you need to obtain a reference to the internal data (possibly for a longer operation) and are okay with runtime borrow checking.

## Common Pitfalls and Compiler Insights

*   **`RefCell<T>` Borrow Violations at Runtime:**
    *   **Pitfall:** Calling `borrow_mut()` while another `borrow()` or `borrow_mut()` is active.
        ```rust
        // use std::cell::RefCell;
        // let data = RefCell::new(String::from("hello"));
        // let r1 = data.borrow();
        // let r2 = data.borrow_mut(); // PANIC! Already borrowed as immutable
        // println!("{}", r1);
        ```
    *   **Runtime Behavior:** The program will panic with a message like `RefCell<T> already mutably borrowed` or `RefCell<T> already borrowed`. This is not a compiler error but a runtime crash.
    *   **Fix:** Ensure that borrows are correctly scoped. `Ref<T>` and `RefMut<T>` smart pointers automatically release the borrow when they go out of scope. Use `try_borrow()` or `try_borrow_mut()` if you want to handle potential borrow conflicts gracefully instead of panicking.

*   **Using `Cell<T>` with Non-`Copy` Types:**
    *   **Pitfall:** Attempting `Cell<String>` or similar.
    *   **Compiler Insight:** The compiler will give an error: `the trait Copy is not implemented for String` (or whatever non-`Copy` type you used).
    *   **Fix:** Use `RefCell<T>` for non-`Copy` types, or if `T` can be `Copy`, ensure it is.

*   **Forgetting `RefCell<T>` is Not Thread-Safe (`Sync`):**
    *   **Pitfall:** Trying to share `Rc<RefCell<T>>` across threads (e.g., by sending it to `thread::spawn`).
    *   **Compiler Insight:** `RefCell<std::string::String> cannot be shared between threads safely` because `the trait Sync is not implemented for RefCell<std::string::String>`. Similarly for `Send` if you try to transfer ownership.
    *   **Fix:** For thread-safe interior mutability, use `Mutex<T>` or `RwLock<T>`, typically wrapped in an `Arc<T>` (e.g., `Arc<Mutex<T>>`).

## Exercises/Challenges

1.  **`Cell<T>` for Configuration:** Create a `Config` struct that holds a `version: Cell<u32>` and a `name: String`. Implement a method `update_version(&self, new_version: u32)` that updates the version using the `Cell`. Why is `Cell` appropriate here for `version` but not directly for `name`?

2.  **`RefCell<T>` for a Logger:** Implement a simple `Logger` struct that has a `messages: RefCell<Vec<String>>` field. Create a method `log(&self, message: &str)` that adds the message to the `Vec`. Demonstrate that you can log messages. Then, add another method `print_logs(&self)` that immutably borrows the messages and prints them. Show what happens if you try to call `log` while `print_logs` has an active borrow (e.g., by calling `log` from within a loop that is also holding onto the result of `messages.borrow()` from `print_logs`).

3.  **`Rc<RefCell<T>>` Observer Pattern:**
    *   Define a `Subject` struct that holds a value and a list of observers: `observers: Vec<Rc<dyn Observer>>`.
    *   Define an `Observer` trait with a method `update(&self, value: i32)`.
    *   Create a `ConcreteObserver` struct that implements `Observer`. Its `update` method should perhaps print the value and its own ID.
    *   The `Subject` should have methods `attach(&mut self, observer: Rc<dyn Observer>)`, `detach(&mut self, observer_id: /* some way to identify observer */)`, and `notify_observers(&self)` which calls `update` on all observers.
    *   Now, the challenge: How can an observer, when its `update` method is called, modify its own internal state if it's only held by an `Rc`? (Hint: The observer itself might need `RefCell` for its state). Demonstrate this by having `ConcreteObserver` count how many times it has been updated.

## Key Takeaways/Summary

*   Interior mutability allows mutating data even through an immutable reference, moving borrow checking from compile time to runtime for specific types.
*   `Cell<T>` is for `Copy` types, offering simple `get`/`set` operations with minimal overhead.
*   `RefCell<T>` is for any type, providing `borrow()` and `borrow_mut()` methods that enforce borrowing rules at runtime (panicking on violation).
*   The combination `Rc<RefCell<T>>` is common for achieving shared, mutable data in single-threaded contexts.
*   Neither `Cell<T>` nor `RefCell<T>` are inherently thread-safe for shared mutation across threads; for that, `Mutex<T>` or `RwLock<T>` (usually with `Arc<T>`) are used.
*   Interior mutability is a powerful tool but should be used judiciously, as runtime panics can be harder to debug than compile-time errors.

---

While `Cell<T>` and `RefCell<T>` provide powerful ways to manage mutability, the combination of `Rc<T>` and `RefCell<T>` can sometimes lead to a subtle problem: reference cycles, which can cause **memory leaks**. We'll explore this and how to prevent it in Part 7.
