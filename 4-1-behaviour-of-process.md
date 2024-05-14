 - [Start Process Method](#start-process-method)
- [Module scope mutable variable](#module-scope-mutable-variable)
  * [Fork](#fork)
  * [Spawn](#spawn)
- [Workers logger](#workers-logger)
  * [Spawn](#spawn-1)
    + [Code modification 1](#code-modification-1)
  * [Fork](#fork-1)
    + [Code modification 1](#code-modification-1-1)
    + [Code Modification 2](#code-modification-2)
- [Data Sink](#data-sink)
  * [Spawn](#spawn-2)
  * [Fork](#fork-2)
  * [Results and Conclusion](#results-and-conclusion)
  * [Solution - Shared Memory](#solution---shared-memory)

Notes:
- <small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>
- <small><i><a href='https://www.youtube.com/watch?v=mASFlnQBUW4'>Test 3 based on Łukasz Langa talk</a></i></small>

---

# Start Process Method
<table><tr><th></th>
	<th scope="col">Fork</th>
	<th scope="col">Spawn</th>
</tr><tr><th scope="row">Memory Overhead</th><td>
Could be higher as it duplicates entire memory space of parent process for child process
</td><td>
Lower, as it creates a new process by launching a new executable and initializing a new runtime environment 
</td></tr><tr><th scope="row">Resource Utilization</th><td>
Yes, copy-on-write typically employed to optimize memory usage
</td><td>
No, copy-on-write not applicable, as it creates a new process with separate memory space
</td></tr><tr><th scope="row">Performance Impact</th><td>
Can be significant, especially for large processes or frequent forking
</td><td>
Generally lower, as it involves less overhead for process creation 
</td></tr><tr><th scope="row">Platform Support & Portability</th><td>
Primarily used in Unix-like systems (e.g., Linux, macOS, BSD) 
</td><td>
Available on various operating systems, including Unix-like systems and Windows 
</td></tr><tr><th scope="row">Resource Management</th><td>
Requires careful management of resources to avoid issues like file descriptor leaks
</td><td>
Generally simpler, as each process has its own separate resources 
</td></tr><tr><th scope="row">Signal Handling</th><td>
Child process inherits signal handlers and signal mask from parent process
</td><td>
Child process does not inherit signal handlers and signal mask from parent process
</td></tr><tr><th scope="row">Security</th><td>
May introduce security vulnerabilities if sensitive information is not properly handled
</td><td>
Generally less prone to security vulnerabilities due to separate memory space |
</td></tr></table>

# Module scope mutable variable

```python
import asyncio
import multiprocessing
import sys
import time
from concurrent.futures import ProcessPoolExecutor
from time import sleep

from rich.console import Console

console = Console(width=500)
print_ts = time.time()


def print(*args, **kwargs) -> None:
    global print_ts
    now = time.time()
    proc = multiprocessing.current_process().name
    if proc == "MainProcess":
        proc = f"[bold]{proc:<16}[/bold]"
    else:
        proc = f"{proc:>16}"
    console.print(
        f"{proc} [[green bold]{now - print_ts:>5.2f}s[/]]",
        *args,
        **kwargs
    )


MODULE_SCOPE_LIST = ["Initial value"]
print(f"[MODULE SCOPE] List(id: {id(MODULE_SCOPE_LIST)}) initially: {MODULE_SCOPE_LIST}")


def append_to_shared_list(count):
    sleep(count)
    print(f"[FUNC SCOPE] List({id(MODULE_SCOPE_LIST)}) initially : {MODULE_SCOPE_LIST}")
    MODULE_SCOPE_LIST.append(f"Child process {count} update")
    print(f"Updated List(id: {id(MODULE_SCOPE_LIST)}) {MODULE_SCOPE_LIST}")


async def main(start_process_method):
    MODULE_SCOPE_LIST.append(f"Parent process update")
    print(
        f"Updated List(id: {id(MODULE_SCOPE_LIST)}): {MODULE_SCOPE_LIST}"
    )

    context = multiprocessing.get_context(start_process_method)
    with ProcessPoolExecutor(max_workers=2, mp_context=context) as executor:
        executor.map(append_to_shared_list, range(1, 3))
    print(f"List(id: {id(MODULE_SCOPE_LIST)}) finally: {MODULE_SCOPE_LIST}")


if __name__ == "__main__":
    start_process_method = sys.argv[1]
    print(f"Start subprocess with '{start_process_method}' method")
    asyncio.run(main(start_process_method))

```

## Fork

```commandline
MainProcess      [ 0.00s] [MODULE SCOPE] List(id: 4395577664) initially: ['Initial value']
MainProcess      [ 0.00s] Start subprocess with 'fork' method
MainProcess      [ 0.00s] Updated List(id: 4395577664): ['Initial value', 'Parent process update']
   ForkProcess-1 [ 1.01s] [FUNC SCOPE] List(4395577664) initially : ['Initial value', 'Parent process update']
   ForkProcess-1 [ 1.01s] Updated List(id: 4395577664) ['Initial value', 'Parent process update', 'Child process 1 update']
   ForkProcess-2 [ 2.01s] [FUNC SCOPE] List(4395577664) initially : ['Initial value', 'Parent process update']
   ForkProcess-2 [ 2.01s] Updated List(id: 4395577664) ['Initial value', 'Parent process update', 'Child process 2 update']
MainProcess      [ 2.02s] List(id: 4395577664) finally: ['Initial value', 'Parent process update']
```
**Consequence of the same memory space & copy-on-write resource utilization**

1. List has the same id for any process.
2. Every fork process contains 'Parent process update'.

## Spawn

```commandline
MainProcess      [ 0.00s] [MODULE SCOPE] List(id: 4362744128) initially: ['Initial value']
MainProcess      [ 0.00s] Start subprocess with 'spawn' method
MainProcess      [ 0.00s] Updated List(id: 4362744128): ['Initial value', 'Parent process update']
  SpawnProcess-1 [ 0.00s] [MODULE SCOPE] List(id: 4399924416) initially: ['Initial value']
  SpawnProcess-2 [ 0.00s] [MODULE SCOPE] List(id: 4388537536) initially: ['Initial value']
  SpawnProcess-1 [ 1.01s] [FUNC SCOPE] List(4399924416) initially : ['Initial value']
  SpawnProcess-1 [ 1.01s] Updated List(id: 4399924416) ['Initial value', 'Child process 1 update']
  SpawnProcess-2 [ 2.01s] [FUNC SCOPE] List(4388537536) initially : ['Initial value']
  SpawnProcess-2 [ 2.01s] Updated List(id: 4388537536) ['Initial value', 'Child process 2 update']
MainProcess      [ 2.11s] List(id: 4362744128) finally: ['Initial value', 'Parent process update']
```
**Consequence of new process launching with a new executable file load into memory & initializing a new runtime environement**

1. List has different id for any process.
2. "[MODULE SCOPE]" log printed three times.
3. None of spawned process contain 'Parent process update'.

# Workers logger

```python 
import asyncio
import logging
import multiprocessing
import sys
from concurrent.futures import ProcessPoolExecutor
from time import sleep

logger = logging.getLogger(__name__)

print(f"{multiprocessing.current_process().name} - {__name__} - Print[MODULE SCOPE]: module scoped reached.")


def call_logger(count):
    sleep(count)
    logger.debug(f"Logger(id: {id(logger)}): function executed.")


def init_logger():
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)

    handler = logging.StreamHandler(sys.stdout)
    handler.setLevel(logging.DEBUG)
    formatter = logging.Formatter('%(processName)s - %(name)s:   %(message)s')
    handler.setFormatter(formatter)
    root.addHandler(handler)


async def main(start_process_method):
    context = multiprocessing.get_context(start_process_method)
    with ProcessPoolExecutor(max_workers=2, mp_context=context) as executor:
        executor.map(call_logger, range(1, 3))


if __name__ == "__main__":
    init_logger()
    start_process_method = sys.argv[1]
    logger.debug(f"Logger(id: {id(logger)}): start subprocess with '{start_process_method}' method")
    asyncio.run(main(start_process_method))

```
## Spawn

```commandline
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4390984784): start subprocess with 'spawn' method
SpawnProcess-2 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
SpawnProcess-1 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
```
- Missing logs for call_log function call

### Code modification 1
Pass init_log function as ProcessPoolExecutor initializer, result:
```text
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4389406160): start subprocess with 'spawn' method
SpawnProcess-1 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
SpawnProcess-2 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
SpawnProcess-1 - __mp_main__:   Logger(id: 4353732432): function executed.
SpawnProcess-2 - __mp_main__:   Logger(id: 4328025936): function executed.
```
- All logs are visible
- Every process has separate logger id

## Fork
```commandline
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4354415696): start subprocess with 'fork' method
ForkProcess-1 - __main__:   Logger(id: 4354415696): function executed.
ForkProcess-2 - __main__:   Logger(id: 4354415696): function executed.
```
- Same logger_ids

### Code modification 1
Move call_log to separate.py module, initialize own logger on module scope. Result:

```commandline
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4360363344): start subprocess with 'fork' method
ForkProcess-1 - separate:   Logger(id: 4367811088): function executed.
ForkProcess-2 - separate:   Logger(id: 4367811088): function executed.
```
- Different logger ids for main and forked processes

### Code Modification 2
Pass init_log function as ProcessPoolExecutor initializer. Result:

```commandline
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4325820432): start subprocess with 'fork' method
ForkProcess-1 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-1 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-2 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-2 - separate:   Logger(id: 4327030224): function executed.
```
- Logs duplication for workers. Contains two handlers - first from parent, second for worker initalizer

# Data Sink
Let's generate some data:
```python
import datetime
import pickle
import random
import sys
import tracemalloc

import numpy as np
import pandas as pd

from mem_trace import print, mem_usage

RECORDS_COUNT = 200_000_000


def generate_data(length: int) -> np.recarray:
    print("Generating data to list.", mem_usage())
    values = [0, 1, np.nan]
    data = [(datetime.datetime.now(), random.choice(values)) for _ in range(length)]
    print("Converting list into data frame.", mem_usage())
    df = pd.DataFrame(data, columns=['timestamp', 'val'])
    np_array = df.to_records()
    return np_array


if __name__ == "__main__":
    tracemalloc.start()
    print(f"Generating {RECORDS_COUNT:_} records of data...")
    data = generate_data(RECORDS_COUNT)
    print(f"Generated {sys.getsizeof(data) / (1024 * 1024):.2f} MB of data. Pickling.", mem_usage())
    with open('data.pickle', 'wb') as f:
        pickle.dump(data, f)
    print("Done.")
```

Let's sum the data:
```python
import asyncio
import multiprocessing
import pickle
import time
import tracemalloc
from concurrent.futures import ProcessPoolExecutor

import numpy as np

from mem_trace import print, mem_usage

WORKERS = 1
BATCH_COUNT = WORKERS


def load_data():
    print("Loading data...")
    with open('pw_asyncio/data.pickle', 'rb') as f:
        result = pickle.load(f)
    return result


def process(data: np.array) -> int:
    print("Received data")
    try:
        return np.nansum(data.val)
    finally:
        print("sending result back.", mem_usage())


async def main(data: np.array) -> None:
    loop = asyncio.get_running_loop()
    total_sum = 0
    start = time.time()
    context = multiprocessing.get_context(sys.argv[1)
    with ProcessPoolExecutor(max_workers=WORKERS, initializer=tracemalloc.start, mp_context=context) as ppe:
        print("Scheduling tasks...", mem_usage())
        batch_size = len(data) // BATCH_COUNT
        batches = [
            loop.run_in_executor(ppe, process, data[i:i + batch_size])
            for i in range(0, len(data), batch_size)
        ]
        print("Waiting for results...", mem_usage())
        done, pending = await asyncio.wait(batches)
    assert len(pending) == 0
    for batch in done:
        total_sum += batch.result()
    total_time_s = time.time() - start
    print(mem_usage())
    print(f"{total_time_s=:.2f} {total_sum=}")


if __name__ == "__main__":
    tracemalloc.start()
    data = load_data()
    print(
        f"Loaded {data.nbytes // 1024 // 1024} MB of data."
        f"[bold cyan]{len(data):_}[/] records"

    )
    print(f"Will use {WORKERS} workers for {BATCH_COUNT} batches.")
    asyncio.run(main(data))

```

Let's run the code for both methods increasing number of workers.

## Spawn
WORKERS = 1
```commandline
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.62s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 1 workers for 1 batches.
MainProcess      [ 0.01s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.00s] Waiting for results... Memory usage: 4577 MB; peak: 4577 MB
  SpawnProcess-1 [10.65s] Received data
  SpawnProcess-1 [ 1.63s] sending result back. Memory usage: 4577 MB; peak: 9155 MB
MainProcess      [12.67s] Memory usage: 4577 MB; peak: 14305 MB
MainProcess      [ 0.00s] total_time_s=12.69 total_sum=66669134.0
```

WORKERS = 4
```commandline
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.54s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 4 workers for 4 batches.
MainProcess      [ 0.01s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.71s] Waiting for results... Memory usage: 5865 MB; peak: 7009 MB
  SpawnProcess-1 [ 2.20s] Received data
  SpawnProcess-1 [ 0.45s] sending result back. Memory usage: 1144 MB; peak: 2288 MB
  SpawnProcess-3 [ 3.37s] Received data
  SpawnProcess-3 [ 0.45s] sending result back. Memory usage: 1144 MB; peak: 2288 MB
  SpawnProcess-2 [ 5.00s] Received data
  SpawnProcess-2 [ 0.48s] sending result back. Memory usage: 1144 MB; peak: 2288 MB
  SpawnProcess-4 [ 6.13s] Received data
  SpawnProcess-4 [ 0.34s] sending result back. Memory usage: 1144 MB; peak: 2288 MB
MainProcess      [ 6.64s] Memory usage: 4577 MB; peak: 7009 MB
MainProcess      [ 0.00s] total_time_s=7.36 total_sum=66669134.0
```

WORKERS = 8
```
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.01s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 8 workers for 8 batches.
MainProcess      [ 0.01s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.28s] Waiting for results... Memory usage: 5221 MB; peak: 5793 MB
  SpawnProcess-1 [ 1.50s] Received data
  SpawnProcess-1 [ 0.19s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-2 [ 1.58s] Received data
  SpawnProcess-2 [ 0.18s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-6 [ 2.20s] Received data
  SpawnProcess-6 [ 0.19s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-5 [ 2.86s] Received data
  SpawnProcess-5 [ 0.23s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-3 [ 3.64s] Received data
  SpawnProcess-3 [ 0.24s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-7 [ 4.39s] Received data
  SpawnProcess-7 [ 0.23s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-4 [ 5.20s] Received data
  SpawnProcess-4 [ 0.22s] sending result back. Memory usage: 572 MB; peak: 1144 MB
  SpawnProcess-8 [ 5.93s] Received data
  SpawnProcess-8 [ 0.15s] sending result back. Memory usage: 572 MB; peak: 1144 MB
MainProcess      [ 6.82s] Memory usage: 4577 MB; peak: 5793 MB
MainProcess      [ 0.00s] total_time_s=7.11 total_sum=66669134.0
```

## Fork
WORKERS = 1
```commandline
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 0.96s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 1 workers for 1 batches.
MainProcess      [ 0.00s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.01s] Waiting for results... Memory usage: 4577 MB; peak: 4577 MB
   ForkProcess-1 [10.22s] Received data
   ForkProcess-1 [ 1.64s] sending result back. Memory usage: 9155 MB; peak: 13733 MB
MainProcess      [12.04s] Memory usage: 4577 MB; peak: 14305 MB
MainProcess      [ 0.00s] total_time_s=12.05 total_sum=66669134.0
```

WORKERS = 4
```
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.53s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 4 workers for 4 batches.
MainProcess      [ 0.00s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.01s] Waiting for results... Memory usage: 4577 MB; peak: 4577 MB
   ForkProcess-1 [ 2.15s] Received data
   ForkProcess-1 [ 0.50s] sending result back. Memory usage: 5722 MB; peak: 6866 MB
   ForkProcess-2 [ 3.40s] Received data
   ForkProcess-2 [ 0.41s] sending result back. Memory usage: 5722 MB; peak: 6866 MB
   ForkProcess-3 [ 4.99s] Received data
   ForkProcess-3 [ 0.44s] sending result back. Memory usage: 5722 MB; peak: 6866 MB
   ForkProcess-4 [ 6.62s] Received data
   ForkProcess-4 [ 0.32s] sending result back. Memory usage: 5722 MB; peak: 6866 MB
MainProcess      [ 6.96s] Memory usage: 4577 MB; peak: 7009 MB
MainProcess      [ 0.00s] total_time_s=6.97 total_sum=66669134.0
```

WORKERS = 8
```
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 0.95s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 8 workers for 8 batches.
MainProcess      [ 0.00s] Scheduling tasks... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.02s] Waiting for results... Memory usage: 4577 MB; peak: 4577 MB
   ForkProcess-1 [ 0.98s] Received data
   ForkProcess-1 [ 0.19s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-2 [ 1.62s] Received data
   ForkProcess-2 [ 0.18s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-3 [ 2.30s] Received data
   ForkProcess-3 [ 0.18s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-4 [ 3.06s] Received data
   ForkProcess-4 [ 0.20s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-5 [ 3.83s] Received data
   ForkProcess-5 [ 0.22s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-6 [ 4.60s] Received data
   ForkProcess-6 [ 0.20s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-7 [ 5.44s] Received data
   ForkProcess-7 [ 0.21s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
   ForkProcess-8 [ 6.21s] Received data
   ForkProcess-8 [ 0.14s] sending result back. Memory usage: 5150 MB; peak: 5722 MB
MainProcess      [ 6.36s] Memory usage: 4577 MB; peak: 5793 MB
MainProcess      [ 0.00s] total_time_s=6.38 total_sum=66669134.0
```


## Results and Conclusion
<table>
    <tr>
        <th></th>
        <th>Spawn</th>
        <th>Fork</th>
    </tr>
    <tr>
        <th>1 Worker</th>
        <td>12.69s</td>
        <td>12.05s</td>
    </tr>
    <tr>
        <th>4 Workers</th>
        <td>7.36s</td>
        <td>6.97s</td>
    </tr>
    <tr>
        <th>8 Workers</th>
        <td>7.11s</td>
        <td>6.38s</td>
    </tr>
</table>

1. Less total time for more than 1 worker.
2. Minor difference between 4 and 8 workers.
3. Logs points concurrency and not parallelism between workers.
4. Receiving data time for both methods is relatively high. It seems there's serialization & deserialization is made every worker run.
5. **Fork methods should have significantly lower time for (4) due to copy-on-write mechanism but it doesn't!**

[Why copy-on-write doesn't work for forking processes.](https://bugs.python.org/issue31558) 

## Solution - Shared Memory

Take a look at changes made on \_\_main\_\_ and process function:

```python
import asyncio
import multiprocessing
import pickle
import time
import tracemalloc
from concurrent.futures import ProcessPoolExecutor
from multiprocessing.managers import SharedMemoryManager
from multiprocessing.shared_memory import SharedMemory

import numpy as np

from mem_trace import print, mem_usage

WORKERS = 8
BATCH_COUNT = WORKERS


def load_data():
    print("Loading data...")
    with open('pw_asyncio/data.pickle', 'rb') as f:
        result = pickle.load(f)
    return result


def process(
        shm_name: str,
        shape: tuple[int, ...],
        dtype: np.dtype,
        offset: int,
        batch_size: int
    ) -> int:
    print("Received data")
    shm = SharedMemory(shm_name)
    data = np.recarray(shape=shape, dtype=dtype, buf=shm.buf)
    print("Received data", mem_usage())
    try:
        return np.nansum(data[offset:offset + batch_size].val)
    finally:
        print("Sending result back.", mem_usage())


async def main(data: np.array, shm_name: str) -> None:
    loop = asyncio.get_running_loop()
    total_sum = 0
    start = time.time()
    context = multiprocessing.get_context(sys.argv[1)
    with ProcessPoolExecutor(max_workers=WORKERS, initializer=tracemalloc.start, mp_context=context) as ppe:
        print("Scheduling tasks...", mem_usage())
        batch_size = len(data) // BATCH_COUNT
        batches = [
            loop.run_in_executor(ppe, process, shm_name, data.shape, data.dtype, i, batch_size)
            for i in range(0, len(data), batch_size)
        ]
        print("Waiting for results...", mem_usage())
        done, pending = await asyncio.wait(batches)
    assert len(pending) == 0
    for batch in done:
        total_sum += batch.result()
    total_time_s = time.time() - start
    print(mem_usage())
    print(f"{total_time_s=:.2f} {total_sum=}")


if __name__ == "__main__":
    tracemalloc.start()
    data = load_data()
    print(
        f"Loaded {data.nbytes // 1024 // 1024} MB of data."
        f"[bold cyan]{len(data):_}[/] records"

    )
    print(f"Will use {WORKERS} workers for {BATCH_COUNT} batches.")
    with SharedMemoryManager() as smm:
        shm = smm.SharedMemory(data.nbytes)
        shm_data = np.recarray(shape=data.shape, dtype=data.dtype, buf=shm.buf)
        print("Coping data to shared memory...", mem_usage())
        np.copyto(shm_data, data)
        print("Copied.", mem_usage())
        del data
        asyncio.run(main(shm_data, shm.name))

```

```commandline
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.57s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 8 workers for 8 batches.
MainProcess      [ 0.22s] Coping data to shared memory... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 3.37s] Copied. Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.10s] Scheduling tasks... Memory usage: 0 MB; peak: 4577 MB
MainProcess      [ 0.01s] Waiting for results... Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-5 [ 0.01s] Received data
   ForkProcess-4 [ 0.01s] Received data
   ForkProcess-2 [ 0.01s] Received data
   ForkProcess-7 [ 0.01s] Received data
   ForkProcess-3 [ 0.01s] Received data
   ForkProcess-4 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-7 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-6 [ 0.01s] Received data
   ForkProcess-3 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-2 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-6 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-5 [ 0.00s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-9 [ 0.01s] Received data
   ForkProcess-8 [ 0.01s] Received data
   ForkProcess-8 [ 0.01s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-9 [ 0.01s] Received data Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-9 [ 1.18s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-8 [ 1.54s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-4 [ 1.58s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-2 [ 1.60s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-3 [ 1.61s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-7 [ 1.61s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-5 [ 1.61s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
   ForkProcess-6 [ 1.62s] Sending result back. Memory usage: 0 MB; peak: 4577 MB
MainProcess      [ 1.63s] Memory usage: 0 MB; peak: 4577 MB
MainProcess      [ 0.00s] total_time_s=0.98 total_sum=66669134.0
```

```commandline
MainProcess      [ 0.00s] Loading data...
MainProcess      [ 1.65s] Loaded 4577 MB of data.200_000_000 records
MainProcess      [ 0.00s] Will use 8 workers for 8 batches.
MainProcess      [ 0.29s] Coping data to shared memory... Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 3.22s] Copied. Memory usage: 4577 MB; peak: 4577 MB
MainProcess      [ 0.10s] Scheduling tasks... Memory usage: 0 MB; peak: 4577 MB
MainProcess      [ 0.02s] Waiting for results... Memory usage: 0 MB; peak: 4577 MB
  SpawnProcess-5 [ 0.02s] Received data
  SpawnProcess-5 [ 0.04s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-3 [ 0.04s] Received data
  SpawnProcess-3 [ 0.05s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-9 [ 0.01s] Received data
  SpawnProcess-8 [ 0.00s] Received data
  SpawnProcess-8 [ 0.05s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-9 [ 0.05s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-7 [ 0.00s] Received data
  SpawnProcess-2 [ 0.00s] Received data
  SpawnProcess-7 [ 0.03s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-2 [ 0.03s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-6 [ 0.00s] Received data
  SpawnProcess-6 [ 0.07s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-4 [ 0.00s] Received data
  SpawnProcess-4 [ 0.07s] Received data Memory usage: 0 MB; peak: 0 MB
  SpawnProcess-5 [ 1.02s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-3 [ 1.00s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-8 [ 1.07s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-9 [ 1.14s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-7 [ 1.15s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-4 [ 1.10s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-6 [ 1.15s] Sending result back. Memory usage: 0 MB; peak: 214 MB
  SpawnProcess-2 [ 1.19s] Sending result back. Memory usage: 0 MB; peak: 214 MB
MainProcess      [ 1.87s] Memory usage: 0 MB; peak: 4577 MB
MainProcess      [ 0.00s] total_time_s=1.37 total_sum=66669134.0
```

- **The total time is around 1s**
- **You can see the actual parallelism by "Sending result back." order.**
- [fork] Peak for every fork process is the same as for main process due to main memory space duplication.
- [spawn] Peak for every spawn process is lower due to own, sperated memory space
- Coping data to shared memory is relatively high, however once done it is present for entire app runtime..
