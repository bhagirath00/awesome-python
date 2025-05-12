# Asyncio Basics: Single-Threaded Cooperative Multitasking

For high-concurrency network tasks—such as a monitoring daemon that queries hundreds of HTTP endpoints or maintains thousands of active connections—spawning OS threads or processes is inefficient due to system memory overhead.

**`asyncio`** is a built-in Python library designed to run concurrent code using a single-threaded **Event Loop** and cooperative multitasking.

In the asyncio model:
1. You define functions as **Coroutines** using the `async def` syntax.
2. Inside a coroutine, you yield control back to the event loop during I/O operations using the **`await`** keyword.
3. The event loop schedules and runs these coroutines, switching tasks when one is waiting for I/O.

In this guide, we will cover coroutine declarations, execution scheduling, and write an asynchronous HTTP status checker using `asyncio.gather`.

---

## The Asyncio Event Loop Flow

```mermaid
sequenceDiagram
    participant Loop as Event Loop
    participant T1 as Coroutine Task 1
    participant T2 as Coroutine Task 2
    
    Loop->>T1: Start execution
    T1->>T1: Run CPU code
    T1->>Loop: await HTTP GET (Yield Control)
    Note over Loop: Task 1 is waiting on network socket
    Loop->>T2: Start execution
    T2->>T2: Run CPU code
    T2->>Loop: await DB Write (Yield Control)
    Note over Loop: Both tasks are waiting on I/O
    Note over Loop: Task 1 Network returns data
    Loop->>T1: Resume Task 1
    T1->>Loop: Return result & Terminate
```

---

## 1. Defining and Running Coroutines

To create a coroutine, prefix the function definition with `async def`. To execute it, use `asyncio.run()`, which initializes the event loop and manages its lifecycle.

```python
import asyncio

async def fetch_data_coroutine():
    print("Starting data fetch...")
    # asyncio.sleep is a non-blocking operation that yields control back to the loop
    # Never use 'time.sleep()' inside async code, as it blocks the entire thread
    await asyncio.sleep(1)
    print("Data fetch completed.")
    return {"status": "ok"}

async def main():
    result = await fetch_data_coroutine()
    print(f"Result: {result}")

if __name__ == "__main__":
    # Start the event loop
    asyncio.run(main())
```

---

## 2. Parallel Execution with `asyncio.gather`

To execute multiple coroutines concurrently, schedule them on the event loop and await their completion using **`asyncio.gather()`**.

Here is an asynchronous mock HTTP endpoint checker:

```python
# Save this file as async_checker.py
import asyncio
import time

async def check_api_endpoint(name, delay_seconds):
    """
    Simulates checking an API endpoint.
    Yields control to the event loop during the simulated delay.
    """
    print(f"▶️ Querying API: {name}...")
    
    # Simulate network latency without blocking the thread
    await asyncio.sleep(delay_seconds)
    
    print(f"✅ Received response from {name} (Latency: {delay_seconds}s)")
    return name, "UP"

async def run_parallel_checks():
    start_time = time.perf_counter()
    
    # 1. Define list of task coroutine objects
    tasks = [
        check_api_endpoint("billing-service", 1.5),
        check_api_endpoint("auth-api", 0.5),
        check_api_endpoint("checkout-service", 1.0),
        check_api_endpoint("notification-api", 2.0)
    ]
    
    print("Scheduling all API checks on the event loop...")
    
    # 2. Gather executes the list of coroutines concurrently
    results = await asyncio.gather(*tasks)
    
    print("\n--- Summary Results ---")
    for name, status in results:
        print(f" - {name:<20}: {status}")
        
    duration = time.perf_counter() - start_time
    # The total duration should equal the longest individual delay (2.0s),
    # not the sum of all delays (5.0s), proving the tasks ran concurrently.
    print(f"\nExecution took: {duration:.2f} seconds")

if __name__ == "__main__":
    asyncio.run(run_parallel_checks())
```

---

## Summary

`asyncio` enables high-concurrency network operations within a single thread. By defining coroutines using `async def`, yielding control during I/O operations with `await`, and executing tasks concurrently with `asyncio.gather`, you can build efficient, non-blocking automation scripts. In the next section, we will cover **asyncio in DevOps pipelines**.
