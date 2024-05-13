- [Processes execption handling and recovering](#processes-execption-handling-and-recovering)
- [Crash tests](#crash-tests)
  * [Recursion](#recursion)
  * [Out of memory](#out-of-memory)
  * [Exception handling & Recovery](#exception-handling-and-recovery)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

---

# Processes execption handling and recovering

```python
import asyncio
import multiprocessing
import sys
import time
from asyncio import Future
from concurrent.futures import ProcessPoolExecutor
from typing import Literal, Callable

from rich.console import Console

WORKERS = 8
WORKER_EXECUTION_TIME_SEC = 60

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


def valid_worker_func():
    print("Hello process! Executing...")
    time.sleep(WORKER_EXECUTION_TIME_SEC)
    print("Done.")
    return 0


class ProcessPool:
    def __init__(self, workers_count: int, start_process_method: Literal["spawn", "fork"]):
        print(f"Will use {WORKERS} workers.")
        print(f"Start new process method: {start_process_method}.")
        context = multiprocessing.get_context(start_process_method)
        self.executor = ProcessPoolExecutor(
            max_workers=workers_count, mp_context=context
        )

    async def run_worker(self, process_function: Callable) -> Future:
        loop = asyncio.get_running_loop()
        try:
            return await loop.run_in_executor(self.executor, process_function)
        except Exception as e:
            print(f"{type(e)}: {e}")
            raise e


async def main():
    print(f"Python version: {sys.version}")
    start_process_method = sys.argv[1]
    if start_process_method not in ("spawn", "fork"):
        raise RuntimeError("available arguments are fork and spawn.")
    process_pool = ProcessPool(WORKERS, start_process_method)

    workers_tasks = [asyncio.create_task(process_pool.run_worker(valid_worker_func)) for _ in range(WORKERS - 1)]


    await asyncio.gather(*workers_tasks)
    print("All tasks finished.")

    for worker_task in workers_tasks:
        if worker_task.cancelled() or worker_task.exception():
            print(f"{worker_task.get_name()} cancelled or failed task.")
            continue
        print(f"{worker_task.get_name()}, result: {worker_task.result()}")


if __name__ == "__main__":
    asyncio.run(main())


```
# Crash tests
Let's try to crash process pool differeny ways.


## Recursion
```python
...
def evil_recursion_worker_func():
    try:
        print(
            "I welcome you, Python, my old friend. But to this place where destiny is made, why have you come "
            "unprepared?"
        )
        evil_recursion_worker_func()
    except:
        print(
            "What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How "
            "could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay "
            "down your weapons. It is not too late for my mercy."
        )
        evil_recursion_worker_func()

...

async def main():
    ...
    workers_tasks.append(asyncio.create_task(process_pool.run_worker(evil_dream)))
    ...
```



- Python 3.9.6
- Darwin Kernel Version 23.4.0
```commandline
MainProcess      [ 0.00s] Will use 8 workers.
MainProcess      [ 0.00s] Start new process method: spawn.
  SpawnProcess-3 [ 0.00s] Hello process! Executing...
  SpawnProcess-4 [ 0.00s] Hello process! Executing...
  SpawnProcess-1 [ 0.00s] Hello process! Executing...
  SpawnProcess-5 [ 0.00s] Hello process! Executing...
  SpawnProcess-2 [ 0.00s] Hello process! Executing...
  SpawnProcess-8 [ 0.00s] Hello process! Executing...
  SpawnProcess-6 [ 0.00s] Hello process! Executing...
  SpawnProcess-7 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-7 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-7 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  ...
  SpawnProcess-7 [ 0.26s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-7 [ 0.26s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  ...
MainProcess      [ 0.43s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [ 0.44s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
```
- Python 3.11.7
- Darwin Kernel Version 23.4.0
```commandline
MainProcess      [ 0.00s] Will use 8 workers.
MainProcess      [ 0.00s] Start new process method: spawn.
  SpawnProcess-1 [ 0.00s] Hello process! Executing...
  SpawnProcess-6 [ 0.00s] Hello process! Executing...
  SpawnProcess-5 [ 0.00s] Hello process! Executing...
  SpawnProcess-7 [ 0.00s] Hello process! Executing...
  SpawnProcess-2 [ 0.00s] Hello process! Executing...
  SpawnProcess-3 [ 0.00s] Hello process! Executing...
  SpawnProcess-4 [ 0.00s] Hello process! Executing...
  SpawnProcess-8 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [ 0.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  ...
  SpawnProcess-8 [ 0.20s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [ 0.20s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [ 0.20s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [ 0.20s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [ 0.20s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  ...
  SpawnProcess-1 [60.01s] Done.
  SpawnProcess-2 [60.00s] Done.
  SpawnProcess-3 [60.01s] Done.
  SpawnProcess-7 [60.00s] Done.
  SpawnProcess-5 [60.01s] Done.
  SpawnProcess-8 [60.00s] Done.
  SpawnProcess-4 [60.01s] Done.
  SpawnProcess-8 [60.99s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [60.99s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-8 [60.00s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [60.01s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  ...
```

## Out of memory
```python
...
WORKER_EXECUTION_TIME_SEC = 60
...

def oom_worker_func():
    print("Hello, Devil!")
    result = [x for x in range(100000000000000)]
    print("Bye Devil!")
    return result

...

async def main():
    ...
    workers_tasks.append(asyncio.create_task(process_pool.run_worker(oom_worker_func)))
    ...
```

- Python 3.9.6 & 3.11.7
- Fork & Spawn
```commandline
MainProcess      [ 0.01s] Will use 8 workers.
MainProcess      [ 0.01s] Start new process method: spawn.
  SpawnProcess-1 [ 0.00s] Hello process! Executing...
  SpawnProcess-6 [ 0.00s] Hello process! Executing...
  SpawnProcess-2 [ 0.02s] Hello process! Executing...
  SpawnProcess-5 [ 0.00s] Hello process! Executing...
  SpawnProcess-4 [ 0.00s] Hello process! Executing...
  SpawnProcess-7 [ 0.00s] Hello process! Executing...
  SpawnProcess-3 [ 0.00s] Hello process! Executing...
  SpawnProcess-8 [ 0.00s] Hello, Devil!
MainProcess      [41.25s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [41.26s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
```

## Exception handling and Recovery
Replace asyncio.gather() with TaskGroup async context manager:

```python
async def main():
    ...
    try:
        async with asyncio.TaskGroup() as tg:
            workers_tasks = [tg.create_task(process_pool.run_worker(valid_worker_func)) for _ in range(WORKERS - 1)]
            workers_tasks.append(tg.create_task(process_pool.run_worker(oom_worker_func)))
    except Exception as e:
        print(f"{type(e)}: {e}")

    # await asyncio.gather(*workers_tasks)
    print("All tasks finished.")
    ...
```

```commandline
MainProcess      [ 0.00s] Will use 8 workers.
MainProcess      [ 0.00s] Start new process method: spawn.
  SpawnProcess-2 [ 0.00s] Hello process! Executing...
  SpawnProcess-4 [ 0.00s] Hello process! Executing...
  SpawnProcess-7 [ 0.00s] Hello process! Executing...
  SpawnProcess-6 [ 0.00s] Hello process! Executing...
  SpawnProcess-5 [ 0.00s] Hello process! Executing...
  SpawnProcess-3 [ 0.00s] Hello process! Executing...
  SpawnProcess-8 [ 0.00s] Hello process! Executing...
  SpawnProcess-1 [ 0.00s] Hello, Devil!
MainProcess      [37.43s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [37.45s] <class 'concurrent.futures.process.BrokenProcessPool'>: A process in the process pool was terminated abruptly while the future was running or pending.
MainProcess      [37.45s] <class 'ExceptionGroup'>: unhandled errors in a TaskGroup (2 sub-exceptions)
MainProcess      [37.45s] All tasks finished.
MainProcess      [37.45s] Task-2 cancelled or failed task.
MainProcess      [37.45s] Task-3 cancelled or failed task.
MainProcess      [37.45s] Task-4 cancelled or failed task.
MainProcess      [37.45s] Task-5 cancelled or failed task.
MainProcess      [37.45s] Task-6 cancelled or failed task.
MainProcess      [37.45s] Task-7 cancelled or failed task.
MainProcess      [37.45s] Task-8 cancelled or failed task.
MainProcess      [37.45s] Task-9 cancelled or failed task.
```


