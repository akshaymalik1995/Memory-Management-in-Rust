# Part 7: Memory Leaks in Rust

## Learning Objectives

*   Understand how memory leaks can occur in Rust, despite its safety features.
*   Identify reference cycles involving `Rc<T>` and `RefCell<T>` as a common cause of leaks.
*   Learn to use `Weak<T>` to break reference cycles and prevent memory leaks.
*   Explore tools and techniques for identifying and debugging memory leaks.

---

Rust's ownership system is designed to prevent many common memory safety issues, including most types of memory leaks where allocated memory is no longer reachable but not deallocated. When an owner goes out of scope, its `drop` method is called, and resources are cleaned up. However, Rust is not entirely immune to memory leaks. The most common way leaks occur in safe Rust is through **reference cycles** created with `Rc<T>` (and `Arc<T>`) combined with interior mutability (like `RefCell<T>`).

## How Can Memory Leaks Happen?

A memory leak occurs when memory is allocated but never deallocated, even though it's no longer needed or accessible by the program. In Rust, this typically doesn't happen because of forgotten `free` calls (as in C/C++), but rather because Rust thinks the memory is still in use.

With `Rc<T>` and `Arc<T>`, memory is deallocated only when the strong reference count drops to zero. If you create a situation where two or more `Rc<T>` instances point to each other in a cycle, and they only hold strong references to each other, their reference counts will never reach zero, even if they are no longer accessible from anywhere else in the program. The items in the cycle keep each other alive.

### Example: A Simple Reference Cycle

Let's consider a scenario where an item can refer to another item, and that other item can refer back.

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    // We use Option because a Node might not point to another Node
    // We use RefCell to allow modification of `next` after Node creation
    next: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("Dropping Node with value: {}", self.value);
    }
}

fn main() {
    // Create node 'a' with value 5, initially pointing to None
    let a = Rc::new(Node {
        value: 5,
        next: RefCell::new(None),
    });

    println!("a initial rc count = {}", Rc::strong_count(&a)); // Count = 1
    println!("a next pointer = {:?}", a.next.borrow());

    // Create node 'b' with value 10, initially pointing to None
    let b = Rc::new(Node {
        value: 10,
        next: RefCell::new(None),
    });

    println!("b initial rc count = {}", Rc::strong_count(&b)); // Count = 1
    println!("b next pointer = {:?}", b.next.borrow());

    // Now, let's make 'a' point to 'b'
    if let Some(link) = a.next.borrow_mut().as_mut() {
        // This path isn't taken as it's None initially
    } else {
        *a.next.borrow_mut() = Some(Rc::clone(&b));
    }

    println!("a rc count after pointing to b = {}", Rc::strong_count(&a)); // Still 1 (a itself)
    println!("b rc count after a points to it = {}", Rc::strong_count(&b)); // Now 2 (b itself + a.next)
    println!("a next pointer = {:?}", a.next.borrow());

    // Now, let's make 'b' point back to 'a', creating a cycle!
    if let Some(link) = b.next.borrow_mut().as_mut() {
        // Not taken
    } else {
        *b.next.borrow_mut() = Some(Rc::clone(&a));
    }

    println!("a rc count after b points to it = {}", Rc::strong_count(&a)); // Now 2 (a itself + b.next)
    println!("b rc count after pointing to a = {}", Rc::strong_count(&b)); // Still 2 (b itself + a.next)
    println!("b next pointer = {:?}", b.next.borrow());

    // What happens if we try to see if the cycle causes issues with dropping?
    // Uncommenting the next line will overflow the stack if we try to print the cycle directly
    // because of recursive Debug printing.
    // println!("a: {:?}", a);

    println!("End of main. a and b are about to go out of scope.");
    // When main ends, `a` and `b` go out of scope.
    // The Rc for `a` is dropped, its strong count goes from 2 to 1.
    // The Rc for `b` is dropped, its strong count goes from 2 to 1.
    // Neither count reaches 0 because `a.next` still holds an Rc to `b`,
    // and `b.next` still holds an Rc to `a`.
    // Thus, their `drop` methods are NOT called, and memory is leaked.
}
```
If you run this code, you will see the `println!` from `main` but you will *not* see the "Dropping Node..." messages for nodes 5 and 10. This is because their strong reference counts never reached zero.

## `Weak<T>`: Breaking Reference Cycles

To prevent reference cycles, Rust provides `Weak<T>`, a **weak reference** counterpart to `Rc<T>` (and `Arc<T>` has `Weak` from `std::sync::Weak`).

Weak references allow you to refer to a value owned by an `Rc<T>` (or `Arc<T>`) without increasing its strong reference count. A `Weak<T>` pointer doesn't express an ownership relationship. It will not prevent the data from being dropped.

To access the value a `Weak<T>` points to, you must *upgrade* it by calling the `upgrade()` method. This returns an `Option<Rc<T>>` (or `Option<Arc<T>>`). You get `Some(Rc<T>)` if the value still exists (i.e., its strong count is > 0), or `None` if the value has already been dropped.

Let's modify our `Node` to use `Weak<T>` for one of its links to break potential cycles. For example, a child node might weakly refer to its parent.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct TreeNode {
    value: i32,
    parent: RefCell<Weak<TreeNode>>, // Parent is a weak reference
    children: RefCell<Vec<Rc<TreeNode>>>,
}

impl Drop for TreeNode {
    fn drop(&mut self) {
        println!("Dropping TreeNode with value: {}", self.value);
    }
}

fn main() {
    let leaf = Rc::new(TreeNode {
        value: 3,
        parent: RefCell::new(Weak::new()), // Initially no parent
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf parent = {:?}, leaf strong = {}, weak = {}",
        leaf.parent.borrow().upgrade().map(|p| p.value),
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf)
    ); // strong = 1, weak = 0

    let branch = Rc::new(TreeNode {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    // Set leaf's parent to branch
    // Rc::downgrade(&branch) creates a Weak pointer from an Rc
    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!(
        "branch strong = {}, weak = {}",
        Rc::strong_count(&branch),
        Rc::weak_count(&branch) // weak = 1 because leaf.parent points to branch
    );
    println!(
        "leaf parent = {:?}, leaf strong = {}, weak = {}",
        leaf.parent.borrow().upgrade().map(|p| p.value),
        Rc::strong_count(&leaf), // strong = 1 (original Rc)
        Rc::weak_count(&leaf)  // weak = 0 (no Weak pointers directly to leaf)
    );

    println!("End of main. branch and leaf are about to go out of scope.");
    // When branch goes out of scope, its strong count becomes 0 (leaf.parent is weak).
    // So, branch is dropped.
    // When branch is dropped, its children Vec is dropped. This drops the Rc<TreeNode> for leaf.
    // Now leaf's strong count becomes 0, so leaf is dropped.
    // No cycle, no leak!
}
```

In this tree example:
*   A parent (`branch`) holds strong references (`Rc`) to its children (`leaf`).
*   A child (`leaf`) holds a weak reference (`Weak`) to its parent (`branch`).

When `branch` goes out of scope at the end of `main`, its strong count becomes 0 because `leaf.parent` only holds a `Weak` reference, which doesn't contribute to the strong count. So, `branch` is dropped. When `branch` is dropped, its `children` vector is dropped, which in turn drops the `Rc<TreeNode>` pointing to `leaf`. This makes `leaf`'s strong count 0, and `leaf` is also dropped. All memory is reclaimed.

**Key points about `Weak<T>`:**
*   Created by calling `Rc::downgrade(&rc_instance)`.
*   Does not increase the strong reference count.
*   Does increase a *weak* reference count. `Rc<T>` will only deallocate its data when `strong_count` is 0, regardless of `weak_count`.
*   To use a `Weak<T>`, call `upgrade()` which returns `Option<Rc<T>>`.
*   Essential for data structures where entities can refer to each other, like graphs or parent-child relationships in a tree, allowing cycles to exist logically without causing memory leaks.

## Identifying and Debugging Memory Leaks

While `Weak<T>` helps prevent cycles, sometimes leaks can still occur or you might suspect them.

1.  **Careful Code Review:** The primary way to find `Rc`/`Arc` cycles is by carefully reviewing code where these types are used, especially when combined with `RefCell` or other interior mutability patterns. Look for patterns where object A can hold an `Rc` to object B, and object B can hold an `Rc` to object A.

2.  **Logging `Drop`:** As shown in the examples, implementing the `Drop` trait and adding a `println!` can help you verify if your objects are being deallocated as expected. If you don't see the drop messages for objects you expect to be cleaned up, you might have a leak.

3.  **Memory Profiling Tools:** For more complex scenarios, external memory profiling tools can be invaluable. The specific tools depend on your operating system:
    *   **Valgrind** (Linux, macOS): While primarily for C/C++, Valgrind's `memcheck` tool can sometimes be used with Rust programs to detect leaks, though it might produce false positives or be noisy due to Rust's memory management.
    *   **Heaptrack** (Linux): Another powerful heap memory profiler for Linux.
    *   **Instruments** (macOS): Part of Xcode, Instruments has allocation and leak detection tools.
    *   **Windows Performance Toolkit (WPT):** Contains tools like `HeapAllocSite` for analyzing heap usage.
    *   **Language-Specific Tools:** Some Rust-specific tools or crates might emerge or exist for memory analysis. For example, crates like `jemallocator` (an alternative allocator) sometimes offer profiling features.

4.  **Testing with `Weak::upgrade()`:** If you suspect a particular `Rc` or `Arc` is part of a leak, you can temporarily store a `Weak` reference to it. Later in your program, try to `upgrade()` this `Weak` reference. If it still upgrades successfully after you expect the object to have been dropped, it indicates the object is still alive, possibly due to a leak.

## Other Potential Sources of Leaks (Less Common in Safe Rust)

*   **Forgetting to join threads (for `Arc`):** If an `Arc` is cloned into a detached thread that runs indefinitely (or longer than the main program expects), the data it points to won't be deallocated until that thread finishes and drops its `Arc`.
*   **`std::mem::forget`:** This function explicitly tells Rust to run a value’s destructor. If you `mem::forget` a `Box<T>` or an `Rc<T>`, the memory it manages will be leaked (unless handled by other means). This is generally used in `unsafe` code or FFI contexts with very specific reasons.
*   **Bugs in `unsafe` code:** If you write `unsafe` Rust code that manually manages memory, you can introduce leaks just like in C/C++.

## Exercises/Challenges

1.  **Detect the Cycle:** Review the following code. Does it create a reference cycle and leak memory? Explain why or why not. If it does, how would you fix it using `Weak`?
    ```rust
    use std::rc::Rc;
    use std::cell::RefCell;

    struct Owner {
        name: String,
        gadget: RefCell<Option<Rc<Gadget>>>,
    }

    struct Gadget {
        id: i32,
        owner: Rc<Owner>, // Potential cycle here?
    }

    impl Drop for Owner {
        fn drop(&mut self) {
            println!("Dropping Owner: {}", self.name);
        }
    }
    impl Drop for Gadget {
        fn drop(&mut self) {
            println!("Dropping Gadget: {}", self.id);
        }
    }

    fn main() {
        let owner_alice = Rc::new(Owner {
            name: "Alice".to_string(),
            gadget: RefCell::new(None),
        });

        let gadget1 = Rc::new(Gadget {
            id: 1,
            owner: Rc::clone(&owner_alice),
        });

        // Alice now owns gadget1
        *owner_alice.gadget.borrow_mut() = Some(Rc::clone(&gadget1));

        println!("Alice's gadget count: {}", Rc::strong_count(&gadget1));
        println!("Gadget1's owner count: {}", Rc::strong_count(&owner_alice));
        // What happens when owner_alice and gadget1 go out of scope?
    }
    ```

2.  **Bidirectional Linked List:** Try to implement a simplified doubly linked list where each node has a `value: i32`, a `next: Option<Rc<Node>>`, and a `prev: Option<Weak<Node>>`. Ensure that when a list or parts of it are dropped, all nodes are properly deallocated (use `Drop` trait with `println!` to verify).

3.  **Observer Pattern with `Weak`:** In Part 6, we discussed an Observer pattern. If an observer needs to unregister itself from a subject, or if the subject might be destroyed while observers still exist, `Weak` references can be useful. Modify the Observer pattern idea: a `Subject` holds `Vec<Weak<dyn Observer>>`. When notifying, it iterates, upgrades the `Weak` pointers, and calls `update` if successful. This prevents the `Subject` from keeping observers alive if they are owned elsewhere and dropped.

## Key Takeaways/Summary

*   Memory leaks in safe Rust are primarily caused by reference cycles involving `Rc<T>` (or `Arc<T>`) and `RefCell<T>` (or other interior mutability types).
*   A cycle occurs when objects hold strong references (`Rc`/`Arc`) to each other, preventing their reference counts from ever reaching zero.
*   `Weak<T>` (from `Rc::downgrade` or `Arc::downgrade`) creates non-owning weak references that don't contribute to the strong count, allowing them to break cycles.
*   To use a `Weak<T>`, you must `upgrade()` it to an `Option<Rc<T>>` (or `Option<Arc<T>>`).
*   Identifying leaks involves code review, `Drop` trait logging, and potentially external memory profiling tools.
*   While Rust prevents many leaks, careful design is needed when using shared ownership with interior mutability.

---

Understanding how to prevent and manage memory leaks is crucial for long-running applications. Next, we'll look at another important aspect of resource management: the **`Drop` trait**, which allows types to define custom cleanup logic when they go out of scope.
