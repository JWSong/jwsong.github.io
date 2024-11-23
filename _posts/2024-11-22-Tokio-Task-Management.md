---
layout: post
title: "Understanding Tokio's Thread Types and Task Management"
description: "How Tokio manages threads and tasks, and how to use them efficiently."
keywords: "rust, tokio, thread, async, concurrency, runtime"
author: "Jungwoo Song"
show_in_post_list: true
include_in_header: false
include_in_footer: false
robots: index, follow
---

Tokio is a powerful runtime for asynchronous Rust applications. It provides a high-level API for building concurrent and distributed applications. At its core, Tokio manages threads and tasks to execute asynchronous code efficiently. This post explores the different types of threads Tokio uses and how to manage tasks effectively.

# Understanding Tokio's Thread Types and Task Management

## Introduction
When building asynchronous applications in Rust with Tokio, understanding how it manages threads and tasks is crucial for optimal performance. Tokio uses two distinct types of threads - core threads and blocking threads - each serving a specific purpose in the runtime.

## Core Threads: The Heart of Async Operations

Core threads are where Tokio runs its async code. By default, Tokio creates one core thread per CPU core, though this can be configured using the `TOKIO_WORKER_THREADS` environment variable.

### How Core Threads Work

Core threads operate on the principle of task switching at `.await` points. When a task reaches an `.await`, Tokio can pause it and switch to another task, enabling concurrent execution without using multiple threads.

Here's a simple example:
```rust
async fn task_1() {
    for i in 0..3 {
        println!("Task 1: Step {}", i);
        // This .await allows other tasks to run
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }
}

async fn task_2() {
    for i in 0..3 {
        println!("Task 2: Step {}", i);
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }
}

#[tokio::main]
async fn main() {
    // These tasks will efficiently interleave on core threads
    let (t1, t2) = tokio::join!(
        tokio::spawn(task_1()),
        tokio::spawn(task_2())
    );
}
```

The output might look like:
```sh
Task 1: Step 0
Task 2: Step 0
Task 1: Step 1
Task 2: Step 1
Task 1: Step 2
Task 2: Step 2
```

## Blocking Threads: Handling Blocking Operations

While core threads are great for async operations, they can become a bottleneck when dealing with blocking operations. This is where blocking threads come in.

### The Problem with Blocking Code

Here's an example of code that would harm performance if run on a core thread:
```rust
async fn good_async_task() {
    for i in 0..3 {
        println!("Good async task Step: {}", i);
        // This .await allows other tasks to run
        tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    }
}

async fn bad_blocking_task() {
    // This will block the core thread
    heavy_computation();
}

fn heavy_computation() {
    // This will block the core thread
    for i in 0..100 {
        println!("Blocking task Step: {}", i);
        // Note that it's not a tokio::time::sleep
        // This simulates a CPU-intensive operation
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}

#[tokio::main(worker_threads = 2)]
async fn main() {
    let (t1, t2, t3) = tokio::join!(
        tokio::spawn(bad_blocking_task()),
        tokio::spawn(bad_blocking_task()),
        tokio::spawn(good_async_task())
    );
}
```

The output might look like:
```sh
Blocking task Step: 0
Good async task Step: 0
Blocking task Step: 0
Blocking task Step: 1
Blocking task Step: 1
Blocking task Step: 2
Blocking task Step: 2
Blocking task Step: 3
Blocking task Step: 3
Blocking task Step: 4
Blocking task Step: 4
Blocking task Step: 5
Blocking task Step: 5
...
```

Once the good async task yields, the blocking tasks will take over the core threads and run to completion. The good async task has to wait until the blocking tasks have completed before it can resume execution.

## Solution: Using Blocking Threads

To avoid blocking the core threads, we can use blocking threads. Blocking threads are separate from core threads and are used to handle blocking operations. Let's see how this works:

```rust
#[tokio::main(worker_threads = 2)]
async fn main() {
    let _t1 = tokio::task::spawn_blocking(|| {
        heavy_computation();
    });

    let _t2 = tokio::task::spawn_blocking(|| {
        heavy_computation();
    });

    good_async_task().await;
}
```

The output might look like:
```sh
Good async task Step: 0
Blocking task Step: 0
Blocking task Step: 0
Blocking task Step: 1
Good async task Step: 1
Blocking task Step: 1
Blocking task Step: 2
Good async task Step: 2
Blocking task Step: 2
Blocking task Step: 3
Blocking task Step: 3
Blocking task Step: 4
Blocking task Step: 4
```

The good async task can run concurrently with the blocking tasks. This is because the blocking threads are separate from the core threads.

## Conclusion

By understanding the difference between core threads and blocking threads, you can write more efficient and responsive asynchronous applications with Tokio. Use blocking threads for blocking operations and core threads for async operations to maximize performance and concurrency.
