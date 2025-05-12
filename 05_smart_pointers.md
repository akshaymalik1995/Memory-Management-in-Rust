# Part 5: Smart Pointers

## Learning Objectives

*   Understand what smart pointers are and why they are useful.
*   Learn to use `Box<T>` for simple heap allocation and ownership.
*   Explore `Rc<T>` for enabling multiple owners of the same data.
*   Discover `Arc<T>` for thread-safe multiple ownership.
*   Grasp the role of the `Deref` and `Drop` traits in how smart pointers behave.

---

In previous parts, we've seen how Rust manages memory through ownership, borrowing, and lifetimes, and how data can live on the stack or the heap. We briefly encountered `Box<T>` as a way to allocate data on the heap. `Box<T>` is an example of a **smart pointer**.

Smart pointers are data structures that act like pointers but also have additional metadata and capabilities. In Rust, they are typically structs that implement the `Deref` and `Drop` traits. The `Deref` trait allows a smart pointer struct to behave like a reference, and the `Drop` trait allows you to customize the code that is run when an instance of the smart pointer goes out of scope.

Smart pointers are a common pattern in systems programming, and Rust leverages them extensively to provide safety and convenience.

## `Box<T>`: Simple Heap Allocation

We've already used `Box<T>` to allocate data on the heap. Its primary use cases are:

1.  **Storing data on the heap:** When you want to explicitly allocate data on the heap instead of the stack.
    ```rust
    fn main() {
        let b = Box::new(5); // The value 5 is allocated on the heap
        println!("b = {}", b);
        // *b gives us the value inside the Box
        println!("value inside b = {}", *b);
    } // b goes out of scope, and the memory for 5 is deallocated
    ```

2.  **Recursive types:** For defining types whose size cannot be known at compile time because they contain themselves. `Box` provides a fixed-size pointer to the heap-allocated recursive part.
    ```rust
    enum List {
        Cons(i32, Box<List>),
        Nil,
    }

    use List::{Cons, Nil};

    fn main() {
        let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
        // This creates a list (1, (2, (3, Nil)))
        // Each Cons cell is on the stack, but it holds a Box pointing to the next List item on the heap.
    }
    ```

3.  **Trait objects:** When you want to use a value whose type is only known at runtime (dynamic dispatch), you often use a trait object like `Box<dyn Trait>`. (We'll cover traits in more depth in a future series, but `Box` is essential here).

`Box<T>` implements `Deref` (so you can treat it like a reference to `T`) and `Drop` (to deallocate the heap memory when it goes out of scope). It enforces Rust's normal ownership rules: one owner, and the data is cleaned up when the owner is dropped.

## `Rc<T>`: Reference Counted Smart Pointer

Sometimes, a single value might need to have multiple owners. For example, in a graph data structure, multiple edges might point to the same node, and that node should only be cleaned up when no more edges point to it.

This is where `Rc<T>` (Reference Counter) comes in. `Rc<T>` keeps track of the number of references to a value. When a new reference is created (by calling `Rc::clone(&rc_value)`), the count increases. When an `Rc<T>` goes out of scope, the count decreases. The data on the heap is only cleaned up when the reference count reaches zero.

```rust
use std::rc::Rc;

enum ListRc {
    Cons(i32, Rc<ListRc>),
    Nil,
}

use ListRc::{Cons as ConsRc, Nil as NilRc};

fn main() {
    let a = Rc::new(ConsRc(5, Rc::new(ConsRc(10, Rc::new(NilRc)))));
    println!("Count after creating a: {}", Rc::strong_count(&a)); // Count is 1

    // We use Rc::clone to create another pointer to the same data.
    // This increases the reference count, it does NOT do a deep copy.
    let b = Rc::clone(&a); // b now also points to the list starting with 5. Count increases.
    println!("Count after creating b: {}", Rc::strong_count(&a)); // Count is 2

    {
        let c = Rc::clone(&a); // c also points to the list. Count increases.
        println!("Count after creating c: {}", Rc::strong_count(&a)); // Count is 3
    } // c goes out of scope. Count decreases.

    println!("Count after c goes out of scope: {}", Rc::strong_count(&a)); // Count is 2

    // Now a and b still refer to the data.
    // The data will only be cleaned up when both a and b go out of scope.
}
```

**Important Notes for `Rc<T>`:**

*   `Rc<T>` is only for use in **single-threaded** scenarios. It does not use any synchronization primitives for updating the reference count, making it faster but not thread-safe.
*   `Rc::clone(&value)` only increments the reference count and gives you a new `Rc<T>` pointer to the same data. It does not perform a deep clone of the underlying data. This is a cheap operation.
*   The data pointed to by `Rc<T>` is immutable. You cannot get a mutable reference to it directly because multiple owners could try to mutate it, leading to data races or inconsistencies. If you need mutability with `Rc<T>`, you'd typically combine it with `Cell<T>` or `RefCell<T>` (covered in Part 6).

## `Arc<T>`: Atomically Reference Counted Smart Pointer

When you need multiple ownership across threads, `Rc<T>` is not suitable. Instead, Rust provides `Arc<T>` (Atomically Reference Counted).

`Arc<T>` is functionally similar to `Rc<T>` but uses atomic operations (which are thread-safe) to manage the reference count. This makes `Arc<T>` safe to share between threads.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let numbers = Arc::new(vec![10, 20, 30]);
    let mut handles = vec![];

    println!("Initial Arc count: {}", Arc::strong_count(&numbers)); // Count is 1

    for i in 0..3 {
        let numbers_clone = Arc::clone(&numbers); // Clone Arc for the new thread
        println!("Arc count before thread {}: {}", i, Arc::strong_count(&numbers_clone));

        let handle = thread::spawn(move || {
            // This thread now has ownership of numbers_clone
            println!("Thread {} sees numbers: {:?}, item: {}", i, numbers_clone, numbers_clone[i]);
            // numbers_clone is dropped when the thread finishes, decreasing the count
        });
        handles.push(handle);
    }

    // Wait for all threads to finish
    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final Arc count: {}", Arc::strong_count(&numbers));
    // The original `numbers` Arc is still alive here.
    // If all clones were dropped and this also goes out of scope, count becomes 0 and data is freed.
}
```

**Important Notes for `Arc<T>`:**

*   Use `Arc<T>` when you need to share ownership of data across multiple threads.
*   Atomic operations have a higher performance cost than the non-atomic operations in `Rc<T>`, so prefer `Rc<T>` if you are certain your code is single-threaded.
*   Like `Rc<T>`, the data pointed to by `Arc<T>` is immutable by default. For interior mutability with `Arc<T>` in a thread-safe way, you'd typically use `Mutex<T>` or `RwLock<T>` (which are synchronization primitives, often combined with `Arc`).

## The `Deref` Trait

The `Deref` trait allows an instance of a smart pointer struct to be treated like a regular reference. If a type `MyBox<T>` implements `Deref<Target=T>`, then code that works with `&T` can also work with `&MyBox<T>` through a process called *deref coercion*.

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T; // Associated type defining what &MyBox<T> dereferences to

    fn deref(&self) -> &Self::Target {
        &self.0 // Returns a reference to the inner data
    }
}

fn display_greeting(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let m = MyBox::new(String::from("Rust"));

    // Without Deref, we would have to write: display_greeting(&(*m)[..]);
    // With Deref, Rust can do deref coercion:
    // &MyBox<String> -> &String (via MyBox::deref)
    // &String        -> &str (via String::deref, as String implements Deref<Target=str>)
    display_greeting(&m);

    // Explicit dereference:
    assert_eq!(*m, String::from("Rust"));
}
```
`Box<T>`, `Rc<T>`, and `Arc<T>` all implement `Deref`.

## The `Drop` Trait

The `Drop` trait allows you to customize what happens when a value goes out of scope. This is how `Box<T>` deallocates its heap memory, and how `Rc<T>`/`Arc<T>` decrement their reference counts.

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data: `{}`!", self.data);
        // Custom cleanup code would go here (e.g., releasing a file handle, a network connection, etc.)
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointers created.");
    // c and d will be dropped when they go out of scope, in reverse order of creation.
    // So, d will be dropped first, then c.
    // You cannot call `c.drop()` explicitly.
    // If you need to force a drop earlier, use `std::mem::drop(value)`.
}
```

## Common Pitfalls and Compiler Insights

*   **Using `Rc<T>` in a multi-threaded context:**
    *   **Pitfall:** Trying to send an `Rc<T>` to another thread or share it across threads.
    *   **Compiler Insight:** The compiler will tell you that `Rc<T>` cannot be sent safely between threads (`the trait Send is not implemented for Rc<...>` or `the trait Sync is not implemented for Rc<...>`).
    *   **Fix:** Use `Arc<T>` instead for thread-safe reference counting.

*   **Trying to get a mutable reference from `Rc<T>` or `Arc<T>` directly:**
    *   **Pitfall:** `let data = Rc::new(String::from("test")); let mut_ref = &mut *data;`
    *   **Compiler Insight:** You'll get an error because `Rc<T>` (and `Arc<T>`) only dereferences to an immutable reference (`&T`). This is to prevent data races if multiple owners exist.
    *   **Fix:** If you need to mutate data owned by an `Rc<T>` or `Arc<T>`, use interior mutability patterns with `Cell<T>`, `RefCell<T>` (for `Rc<T>`) or `Mutex<T>`, `RwLock<T>` (for `Arc<T>`). We'll cover `Cell` and `RefCell` in the next part.

*   **Forgetting `Rc::clone` or `Arc::clone` and accidentally moving the smart pointer:**
    *   **Pitfall:**
        ```rust
        // use std::rc::Rc;
        // let rc1 = Rc::new(5);
        // let rc2 = rc1; // Moves rc1, rc1 is no longer valid
        // println!("{}", rc1); // Error!
        ```
    *   **Compiler Insight:** "use of moved value" error.
    *   **Fix:** Always use `Rc::clone(&rc_value)` or `Arc::clone(&arc_value)` to increment the reference count and get a new pointer instance.

*   **Creating reference cycles with `Rc<T>` (and `RefCell<T>`):**
    *   **Pitfall:** If two `Rc` instances point to each other (directly or indirectly) and also contain `RefCell`s or other types that might try to modify them, they can create a cycle where their reference counts never drop to zero, leading to a memory leak.
    *   **Compiler Insight:** The compiler won't catch this logical error. The program will run but leak memory.
    *   **Fix:** Use `Weak<T>` (a weak reference from `Rc`) to break cycles. We'll discuss this in Part 7 on memory leaks.

## Exercises/Challenges

1.  **`Box<T>` for Trait Objects:** Imagine you have a trait `Drawable` with a method `draw(&self)`. Create two different structs that implement `Drawable`. Then, create a `Vec` that can hold instances of any type implementing `Drawable` (Hint: you'll need `Box<dyn Drawable>`). Iterate through the vector and call `draw()` on each element.

2.  **`Rc<T>` for Shared Data:** Create a `User` struct that has an `id: usize` and `name: String`. Create another struct `Group` that has a `name: String` and a `members: Vec<Rc<User>>`. Write code to create a few users and a group. Add some users to the group. Demonstrate that multiple `Rc<User>` pointers can exist and that the user data is shared.

3.  **`Arc<T>` for Threading:** Modify the `User` and `Group` example from exercise 2. Instead of `Rc<User>`, use `Arc<User>`. Create a group and add users. Then, spawn a few threads. Each thread should receive a clone of an `Arc<User>` from the group and print the user's name and ID. Ensure the main thread waits for all spawned threads to complete.

## Key Takeaways/Summary

*   Smart pointers (`Box<T>`, `Rc<T>`, `Arc<T>`) are structs that act like pointers but provide additional capabilities like automatic memory management and shared ownership.
*   `Box<T>` provides simple heap allocation with single ownership.
*   `Rc<T>` allows multiple owners of data in a single-threaded context through reference counting.
*   `Arc<T>` allows multiple owners of data in a multi-threaded context using atomic reference counting.
*   The `Deref` trait enables smart pointers to be treated like references, and `Drop` allows custom cleanup logic when they go out of scope.
*   Understanding smart pointers is crucial for managing heap data, sharing data, and working with dynamic types in Rust.

---

While `Rc<T>` and `Arc<T>` allow sharing immutable data, what if we need to modify data that has multiple owners or is seemingly immutable? This leads us to the concept of **interior mutability**, which we will explore in Part 6.
