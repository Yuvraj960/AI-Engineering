# Module 11: Concurrency – Multithreading, Multiprocessing, and AsyncIO

Welcome to **Module 11: Concurrency – Multithreading (MT), Multiprocessing (MP), and AsyncIO**. In **AI Engineering**, performance bottlenecks dictate whether a system scales smoothly or fails. AI systems continuously deal with two distinct operational bottlenecks:
1. **I/O-bound bottlenecks**: Fetching web data, streaming LLM API tokens (OpenAI, Anthropic), or reading massive image files from cloud storage.
2. **CPU-bound bottlenecks**: Feature engineering, custom mathematical operations, data transformations, and model pre/post-processing.

This module explores Python’s execution models, the **Global Interpreter Lock (GIL)**, synchronization mechanisms, process-based IPC, and single-threaded cooperative event loops via `asyncio`.

---

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Evaluate** the key architectural differences between Parallelism (True Simultaneous Execution) and Concurrency (Interleaved Task Management).
2. **Analyze** the impact of CPython’s **Global Interpreter Lock (GIL)** on multithreaded workloads.
3. **Implement** thread-based concurrency using `threading` and synchronization primitives (`Lock`, `Semaphore`) for I/O-bound tasks.
4. **Bypass the GIL** using `multiprocessing` to run CPU-bound AI/ML data processing pipelines across multiple core processors.
5. **Construct** asynchronous non-blocking workflows using `asyncio` (`async`/`await`, Event Loops, `asyncio.gather()`).
6. **Design** data structures for Inter-Process Communication (IPC) using `multiprocessing.Queue` and `multiprocessing.Pipe`.
7. **Select** the appropriate concurrency strategy (MT vs. MP vs. AsyncIO) for production deployment scenarios.

---

## 🧠 Core Topics & Deep-Dive Concepts

### 1. Concurrency vs. Parallelism & The Global Interpreter Lock (GIL)

* **Concurrency**: Managing multiple execution tasks by interleaving execution on a single core or across multiple cores. (Task switching).
* **Parallelism**: Executing multiple computational tasks simultaneously across physical CPU cores.

```
Single Core (Concurrency via Context Switching):
Core 1: | Task A | Task B | Task A | Task C | Task B |

Multi-Core (Parallelism via True Simultaneous Execution):
Core 1: | ======= Task A ======= |
Core 2: | ======= Task B ======= |
```

#### CPython's Global Interpreter Lock (GIL)
The **GIL** is a mutual exclusion lock held by CPython to ensure that only **one native thread executes Python bytecode at any given instant**.

* **Impact on CPU-bound operations**: Standard multithreading (`threading`) does **not** accelerate CPU-heavy operations (e.g., matrix calculations or loops). In fact, context-switching overhead can make it slower.
* **Impact on I/O-bound operations**: Threads release the GIL while waiting for disk reads, network requests, or database queries. Multithreading is highly effective for I/O operations.

---

### 2. Multithreading (`threading`) & Race Conditions

Multithreading creates OS-level threads within a **single process**, sharing the same memory heap.

#### Race Conditions and Synchronization Primitives (`Lock`)
Because memory is shared, two threads modifying a variable at the same time can corrupt data. A **Mutex (`threading.Lock`)** ensures mutually exclusive access.

```python
import threading
import time

class SharedCounter:
    def __init__(self):
        self.value = 0
        self.lock = threading.Lock()

    def increment(self):
        with self.lock:  # Acquires lock, guaranteeing thread safety
            current = self.value
            time.sleep(0.001)  # Simulates small processing context switch
            self.value = current + 1

counter = SharedCounter()
threads = [threading.Thread(target=counter.increment) for _ in range(10)]

for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Thread-Safe Final Count: {counter.value}")  # Always 10
```

---

### 3. Multiprocessing (`multiprocessing`) & Inter-Process Communication (IPC)

To execute code in parallel across physical CPU cores and bypass the GIL, Python provides the `multiprocessing` module. Each process runs its own CPython interpreter instance with its own isolated memory space.

#### Inter-Process Communication: `Queue`
Because processes do not share memory, data must be serialized (pickled) and passed using IPC structures such as `multiprocessing.Queue`.

```python
import multiprocessing as mp
import math

def cpu_heavy_feature_extraction(numbers: list[int], output_queue: mp.Queue):
    """CPU-bound task running in a separate OS process."""
    results = [math.factorial(n) for n in numbers]
    output_queue.put(results)

if __name__ == "__main__":
    queue = mp.Queue()
    data = [10000, 12000, 15000]

    # Spawn process
    process = mp.Process(target=cpu_heavy_feature_extraction, args=(data, queue))
    process.start()

    # Collect result from queue
    process_results = queue.get()
    process.join()

    print(f"Processed {len(process_results)} large factorials via Multiprocessing.")
```

---

### 4. Asynchronous Programming (`asyncio`)

`asyncio` implements **single-threaded cooperative multitasking** using an **Event Loop**.

#### Key Terminology
* **Event Loop**: Managing loop mechanism executing queued asynchronous tasks.
* **Coroutine (`async def`)**: Specialized functions that pause execution at `await` yield points, returning control to the event loop.
* **`await`**: Relinquishes execution control back to the event loop until the waited operation completes.

```python
import asyncio
import time

async def fetch_llm_response(prompt_id: int) -> str:
    print(f"[API call {prompt_id}] Initiated request...")
    # Non-blocking async sleep simulating network I/O
    await asyncio.sleep(1.5) 
    print(f"[API call {prompt_id}] Received token response.")
    return f"Response for prompt {prompt_id}"

async def main():
    start_time = time.perf_counter()
    
    # Schedule multiple coroutines concurrently on the event loop
    tasks = [fetch_llm_response(i) for i in range(1, 4)]
    results = await asyncio.gather(*tasks)
    
    elapsed = time.perf_counter() - start_time
    print(f"All API calls completed in {elapsed:.2f}s (Concurrent Non-blocking!)")

# Run event loop
asyncio.run(main())
```

---

## 📊 Comprehensive Strategy Comparison Matrix

| Feature / Metric | Multithreading (`threading`) | Multiprocessing (`multiprocessing`) | Asynchronous (`asyncio`) |
| :--- | :--- | :--- | :--- |
| **Execution Model** | Preemptive Multitasking | True Parallel OS Processes | Cooperative Multitasking |
| **GIL Impact** | Subject to GIL restrictions | Bypasses GIL completely | Operates inside single GIL process |
| **Primary Use Case** | Legacy/Blocking I/O bound tasks | Heavy CPU calculations/Data prep | High-throughput Non-blocking I/O |
| **Memory Footprint** | Shared (Low memory overhead) | Isolated memory (High overhead) | Single process (Lowest memory) |
| **Context Switching** | OS-level context switching | OS-level process context switching | Application-level event yield |
| **AI Engineering Role**| File downloads, basic DB calls | Image pre-processing, matrix math | Streaming LLM API calls, Web Scraping |

---

## 🛠️ Practical Hands-On Project: High-Throughput AI Pipeline

This production script combines **AsyncIO** for rapid streaming API data collection, **Multiprocessing** for CPU-bound feature extraction, and **Multithreading** for thread-safe concurrent logging.

```python
# filename: hybrid_concurrency_pipeline.py

import asyncio
import multiprocessing as mp
import threading
import time
import math
from typing import List, Dict, Any

# ==========================================
# 1. Thread-Safe Logging System (Multithreading)
# ==========================================

class ThreadSafeLogger:
    """Thread-safe logging utility using Mutex Locks."""
    def __init__(self):
        self._lock = threading.Lock()

    def log(self, stage: str, message: str):
        with self._lock:
            timestamp = time.strftime("%H:%M:%S")
            print(f"[{timestamp}] [{threading.current_thread().name}] [{stage}] {message}")

logger = ThreadSafeLogger()


# ==========================================
# 2. CPU-Bound Feature Extraction (Multiprocessing)
# ==========================================

def cpu_heavy_transformation(raw_sample: Dict[str, Any]) -> Dict[str, Any]:
    """Simulates CPU-intensive feature computation (e.g., embeddings or audio parsing)."""
    val = raw_sample["raw_val"]
    # Synthetic CPU computation loop
    computed_metric = sum([math.sin(val) * math.cos(val) for _ in range(500000)])
    
    return {
        "sample_id": raw_sample["id"],
        "metric": round(computed_metric, 4),
        "processed_by_pid": mp.current_process().pid
    }

def run_cpu_worker_pool(raw_data: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """Distributes work across CPU cores via a Process Pool."""
    logger.log("CPU Pipeline", f"Spawning worker pool for {len(raw_data)} samples...")
    with mp.Pool(processes=mp.cpu_count()) as pool:
        processed_results = pool.map(cpu_heavy_transformation, raw_data)
    return processed_results


# ==========================================
# 3. Async API Fetcher Engine (AsyncIO)
# ==========================================

async def simulate_async_api_ingestion(sample_id: int) -> Dict[str, Any]:
    """Simulates fetching data from remote LLM endpoints asynchronously."""
    await asyncio.sleep(0.5)  # Non-blocking network I/O simulation
    return {"id": sample_id, "raw_val": sample_id * 1.5}

async def run_async_ingestion_stage(batch_size: int) -> List[Dict[str, Any]]:
    """Collects batches of raw API payloads concurrently."""
    logger.log("Async Pipeline", f"Initiating async network requests for {batch_size} endpoints...")
    tasks = [simulate_async_api_ingestion(i) for i in range(1, batch_size + 1)]
    results = await asyncio.gather(*tasks)
    logger.log("Async Pipeline", "Successfully fetched all API payloads.")
    return results


# ==========================================
# 4. Pipeline Execution Orchestrator
# ==========================================

def main():
    total_start = time.perf_counter()
    print("=" * 65)
    print("🚀 HYBRID CONCURRENCY AI DATA PIPELINE STARTING")
    print("=" * 65)

    # STAGE 1: Async Data Ingestion (I/O Bound)
    raw_payloads = asyncio.run(run_async_ingestion_stage(batch_size=8))

    # STAGE 2: Parallel CPU Feature Processing (CPU Bound)
    cpu_start = time.perf_counter()
    transformed_data = run_cpu_worker_pool(raw_payloads)
    cpu_duration = time.perf_counter() - cpu_start

    # STAGE 3: Final Output Verification
    print("\n--- Pipeline Execution Summary ---")
    for sample in transformed_data:
        print(f"Sample #{sample['sample_id']} | Metric: {sample['metric']} | Process PID: {sample['processed_by_pid']}")

    total_duration = time.perf_counter() - total_start
    print("=" * 65)
    print(f"⏱️ CPU Processing Time : {cpu_duration:.2f}s")
    print(f"⏱️ Total Execution Time : {total_duration:.2f}s")
    print("=" * 65)

if __name__ == "__main__":
    main()
```

---

## ❓ Technical Interview & Conceptual Exam Questions

1. **Why does CPython use the Global Interpreter Lock (GIL), and how can you circumvent it for CPU-heavy tasks?**
   * **Answer:** CPython uses the GIL to ensure thread safety when managing reference counting for garbage collection. You can circumvent the GIL by using the `multiprocessing` module (spawning isolated Python processes), running C extensions (e.g., NumPy/PyTorch) that release the GIL during matrix computations, or using GIL-free Python runtimes (e.g., Python 3.13+ free-threading builds).

2. **What is the difference between Preemptive and Cooperative multitasking?**
   * **Answer:** **Preemptive multitasking** (used by `threading` and OS processes) allows the OS scheduler to forcefully interrupt and switch between threads at any time. **Cooperative multitasking** (used by `asyncio`) relies on code explicitly yielding control (via `await` keywords) back to an event loop.

3. **What happens if a CPU-bound blocking function is executed inside an `asyncio` coroutine?**
   * **Answer:** The CPU-bound function blocks the single-threaded event loop, preventing all other scheduled coroutines from running until the calculation finishes. To handle blocking tasks in `asyncio`, offload them using `asyncio.to_thread()` or `loop.run_in_executor()`.

4. **When should you choose `asyncio` over `threading` for I/O-bound web scrapers?**
   * **Answer:** `asyncio` is preferred when managing thousands of concurrent network connection sockets because it avoids the memory footprint and context-switching overhead of OS-level threads. `threading` is useful when interacting with legacy synchronous libraries that do not support non-blocking `async`/`await` calls.

---

## 📚 Curated Resources & References

### Official Documentation
* [Python Docs – `threading` Module](https://docs.python.org/3/library/threading.html)
* [Python Docs – `multiprocessing` Module](https://docs.python.org/3/library/multiprocessing.html)
* [Python Docs – `asyncio` Asynchronous I/O](https://docs.python.org/3/library/asyncio.html)

### Technical Guides & Advanced Deep Dives
* [Real Python – Speed Up Your Python Program With Concurrency](https://realpython.com/python-concurrency/)
* [Real Python – Async IO in Python: A Complete Walkthrough](https://realpython.com/async-io-python/)
* [Python Docs – Global Interpreter Lock (GIL) Concepts](https://docs.python.org/3/glossary.html#term-global-interpreter-lock)