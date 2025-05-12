# Part 4: Memory Allocation: Stack vs. Heap

## Learning Objectives

*   Understand the difference between stack and heap memory.
*   Learn how Rust determines whether to allocate data on the stack or the heap.
*   Recognize the performance implications of stack vs. heap allocation.
*   See how ownership and borrowing interact with stack and heap data.

---

In our previous discussions on ownership, borrowing, and lifetimes, we touched upon how Rust manages memory. A crucial aspect of this management is *where* in memory data is stored. Programs typically have two main regions of memory available for use during runtime: the **stack** and the **heap**.

Understanding how Rust utilizes these two regions is key to writing efficient and correct code.

## The Stack

Think of the stack as a pile of books. When you add a new book, you place it on top. When you remove a book, you take it from the top. This is a Last-In, First-Out (LIFO) principle.

In programming, the stack is a region of memory where data is stored and retrieved in this LIFO manner. When a function is called, a block of memory (called a *stack frame*) is allocated on the stack for its local variables and some bookkeeping data. When the function finishes, its stack frame is deallocated.

**Characteristics of Stack Allocation:**

*   **Fast Allocation/Deallocation:** Pushing to or popping from the stack is very fast because it just involves incrementing or decrementing a single pointer (the stack pointer).
*   **Fixed Size:** All data stored on the stack must have a known, fixed size at compile time. If the size of a type is unknown at compile time, or if its size might change, it must be stored on the heap instead.
*   **LIFO Order:** Data is accessed in the reverse order it was added.
*   **Automatic Management:** The compiler manages stack allocation and deallocation automatically as functions are called and return.

**What Rust Stores on the Stack:**

*   **Primitive types:** `i32`, `f64`, `bool`, `char`.
*   **Pointers/References:** `&T`, `&mut T`, `Box<T>` (the pointer itself is on the stack, though `Box<T>` points to heap data).
*   **Structs and Enums composed entirely of stack-allocated types:** If all fields of a struct or variants of an enum are themselves stack-allocated types, the struct/enum instance can live on the stack.
*   **Fixed-size arrays:** `[i32; 3]`.
*   **Function call frames:** Local variables, arguments, return addresses.

```rust
fn main() {
    let x = 5;          // i32, stored on the stack
    let y = true;       // bool, stored on the stack
    let arr = [1, 2, 3]; // array of i32s, stored on the stack

    // When some_function is called, a new stack frame is created for it.
    let sum = some_function(x, arr[0]); // x and arr[0] are copied to some_function's stack frame
    println!("Sum: {}", sum);
} // sum, arr, y, x go out of scope. Their memory on the stack is reclaimed.

fn some_function(a: i32, b: i32) -> i32 { // a and b are on some_function's stack frame
    let result = a + b; // result is on some_function's stack frame
    result
} // result, b, a go out of scope. some_function's stack frame is reclaimed.
```

## The Heap

Think of the heap as a more disorganized space, like a restaurant where you ask for a table. The restaurant finds an empty table that fits your party, gives you its location, and you use it. When you're done, you tell the restaurant, and the table becomes available again.

In programming, the heap is a region of memory used for data that can grow or shrink in size at runtime, or whose lifetime is not tied to a particular function call.

**Characteristics of Heap Allocation:**

*   **Slower Allocation/Deallocation:** Requesting memory from the heap involves more work. The system needs to find a block of memory large enough, do some bookkeeping, and return a pointer to it. Deallocation also involves bookkeeping.
*   **Variable Size:** Data stored on the heap can have a size that is determined at runtime or can change.
*   **Flexible Lifetime:** Data on the heap can live longer than the function that created it, as long as something still owns it or has a valid reference to it.
*   **Manual/Systematic Management:** In languages like C/C++, heap memory is managed manually (`malloc`, `free`). In Rust, heap allocation is managed through types like `Box<T>`, `String`, and `Vec<T>`, which handle deallocation automatically when they go out of scope (due to the `Drop` trait).

**What Rust Stores on the Heap:**

*   **Dynamically Sized Types (DSTs):** Data whose size isn't known at compile time, like the contents of a `String` or the elements of a `Vec<T>`.
*   **Data you explicitly box:** When you use `Box::new(value)`, the `value` is moved to the heap, and the `Box<T>` (which is a smart pointer on the stack) points to it.

```rust
fn main() {
    // s1 is a String. It has three parts:
    // 1. A pointer to the actual character data ("hello") on the heap.
    // 2. A capacity (how much memory is allocated on the heap).
    // 3. A length (how many bytes are currently used).
    // These three parts (ptr, capacity, length) are stored on the stack for s1.
    let s1 = String::from("hello"); // Allocates memory on the heap for "hello"

    // v is a Vec<i32>. Similar to String, it has a pointer to heap data,
    // capacity, and length, all stored on the stack for v.
    let v = vec![1, 2, 3]; // Allocates memory on the heap for the integers

    // b is a Box<i32>. The i32 value (5) is stored on the heap.
    // b itself (the pointer to the heap data) is on the stack.
    let b = Box::new(5);

    println!("s1: {}, v: {:?}, b: {}", s1, v, b);

} // s1, v, and b go out of scope.
  // Their `drop` methods are called, which deallocates the heap memory they manage.
  // The stack parts of s1, v, and b are also reclaimed.
```

## When Data Goes on the Stack vs. the Heap

Rust primarily decides based on the type of the data:

*   **Known Size at Compile Time:** If the type's size is known at compile time (e.g., `i32`, `bool`, `[u8; 16]`, a struct with all fixed-size fields), Rust will try to store it on the stack. This is because stack allocation is very fast.

*   **Unknown Size at Compile Time or Dynamic Size:** If the type's size is not known at compile time (e.g., `str` slice, though you usually have `&str` which is a fat pointer of known size) or if it needs to grow (e.g., `String`, `Vec<T>`), the actual data is stored on the heap. A pointer (and potentially length/capacity metadata) to this heap data is stored on the stack.

*   **Explicit Heap Allocation:** You can force data that would normally be on the stack onto the heap using `Box<T>`.
    ```rust
    let five = 5; // on the stack
    let boxed_five = Box::new(5); // 5 is now on the heap, boxed_five (the pointer) is on the stack
    ```
    This is useful for:
    *   Recursive types where the size can't be known (e.g., a cons list node pointing to another node).
    *   Transferring ownership of large amounts of data without copying it (though moves already do this efficiently for heap-allocated types like `String`).
    *   Trait objects (which we haven't covered yet) that require a known size for the pointer.

## Performance Implications

*   **Stack Access is Faster:** Accessing data on the stack is generally faster than accessing data on the heap. This is due to several reasons:
    *   **Allocation/Deallocation Speed:** As mentioned, stack operations are trivial.
    *   **CPU Caching:** Data on the stack is often more cache-friendly because it's typically allocated and accessed in a contiguous block (the current stack frame). Heap data can be fragmented across memory, leading to more cache misses.
    *   **No Indirection (usually):** For stack-allocated values, you have the value directly. For heap-allocated values, you first have a pointer on the stack, which you then dereference to get to the actual data on the heap. This indirection adds a small overhead.

*   **Heap Allocation Overhead:** Allocating memory on the heap has a cost. The allocator needs to find a suitable block of memory, and this can take time, especially if the heap is fragmented or if synchronization is needed in a multithreaded context.

Because of these performance characteristics, Rust (and C++/C) programmers often prefer stack allocation when possible. Rust's ownership system and types like `String` and `Vec<T>` manage heap allocations carefully to mitigate their costs, but the fundamental difference remains.

## Ownership, Borrowing, Stack, and Heap

*   **Stack Data and `Copy`:** Types that are entirely on the stack and have a known, fixed size can implement the `Copy` trait. When you assign a `Copy` type, the bits are simply copied. No heap allocation is involved, and ownership rules are simpler (the original variable remains valid).
    ```rust
    let x = 10; // on stack
    let y = x;  // y is a copy of x, also on stack. x is still valid.
    ```

*   **Heap Data and `Move`:** Types that manage data on the heap (like `String`, `Vec<T>`, `Box<T>`) do not implement `Copy` by default. When you assign them, ownership is moved. The pointer and metadata (length/capacity) on the stack are copied, but the heap data itself is not. The original variable becomes invalid to prevent double-free errors.
    ```rust
    let s1 = String::from("data"); // s1 (stack part) points to "data" (heap part)
    let s2 = s1;                   // s1's pointer/meta is copied to s2. s1 is now invalid.
                                   // No new heap allocation for "data". s2 now owns it.
    // println!("{}", s1); // Compile error: use of moved value
    ```
    If you want a true deep copy of heap data, you use `clone()`:
    ```rust
    let s1 = String::from("data");
    let s2 = s1.clone();           // New heap allocation for "data". s1 and s2 both valid.
    ```

## Common Pitfalls and Compiler Insights

*   **Trying to create dynamically sized data directly on the stack:**
    *   **Pitfall:** Attempting something like `let s: str = "hello";` (this isn't quite how you'd hit it, but the idea is a type whose size isn't known at compile time without a pointer).
    *   **Compiler Insight:** Rust will usually guide you. For `str`, it will say `the size for values of type str cannot be known at compilation time`. It often suggests using a reference (`&str`) or a boxed type (`Box<str>`).
    *   **Fix:** Use types that manage dynamic data (like `String` for growable strings) or references/pointers to unsized data (like `&str`).

*   **Excessive Heap Allocation:**
    *   **Pitfall:** Unnecessarily boxing small values or frequently creating and destroying heap-allocated objects in performance-sensitive loops.
    *   **Compiler Insight:** The compiler won't directly warn about this as it's often valid code, but performance profiling tools can reveal it.
    *   **Fix:** Prefer stack allocation where possible. Reuse buffers (e.g., `String::clear()` and reuse vs. new `String`) if appropriate. Consider if `Copy` types or references are sufficient.

*   **Forgetting `Box<T>` for recursive struct definitions:**
    *   **Pitfall:**
        ```rust
        // struct Node {
        //     value: i32,
        //     next: Node, // Error: recursive type has infinite size
        // }
        ```
    *   **Compiler Insight:** `error[E0072]: recursive type ... has infinite size`.
    *   **Fix:** Use `Box<Node>` to store the `next` node on the heap, breaking the infinite size recursion because `Box<Node>` has a known size (it's a pointer).
        ```rust
        struct Node {
            value: i32,
            next: Option<Box<Node>>,
        }
        ```

## Exercises/Challenges

1.  **Stack or Heap?** For each of the following variable declarations, state whether the primary data it represents is stored on the stack or the heap. If it involves both (like a `String`), explain which part is where.
    *   `let a = 42.0f32;`
    *   `let b = "temporary";`
    *   `let c = String::from("changeable");`
    *   `let d = vec![0u8; 1024];`
    *   `let e = Box::new([1, 2, 3, 4, 5]);`
    *   `struct MyStruct { x: i32, y: bool } let f = MyStruct { x: 10, y: false };`

2.  **Performance Consideration:** You are writing a function that processes a large list of coordinate pairs `(f64, f64)`. You have two options for representing a pair:
    *   A struct `Point { x: f64, y: f64 }`.
    *   A `Box<Point>`.
    If you have a `Vec` of these pairs, which representation (`Vec<Point>` or `Vec<Box<Point>>`) would likely be more performant for iterating through and reading the coordinates? Why? What if you were frequently reordering (sorting) a large `Vec` of these pairs?

3.  **Fix the Recursion:** The following enum is intended to represent a list of integers, but it won't compile. Explain why and fix it.
    ```rust
    // enum IntList {
    //     Cons(i32, IntList),
    //     Nil,
    // }
    ```

## Key Takeaways/Summary

*   The **stack** is fast, LIFO, and for data with a known, fixed size at compile time. Managed automatically.
*   The **heap** is more flexible, for dynamically sized data or data with a longer lifetime. Allocation/deallocation is slower and managed by owning types like `Box`, `String`, `Vec`.
*   Rust prefers stack allocation for speed but uses heap allocation when necessary, managing it safely via the ownership system.
*   Understanding stack vs. heap helps in reasoning about performance and memory layout.
*   `Copy` types are usually stack-only. `Move` semantics are default for heap-managing types to ensure safety and efficiency.

---

Now that we understand where data lives, we can delve deeper into the tools Rust provides for managing heap-allocated data beyond simple ownership. In the next part, we'll explore **smart pointers**, which are structs that act like pointers but also have additional metadata and capabilities.
