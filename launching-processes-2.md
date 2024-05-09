1. [Lazy loading and collection generation peak](https://github.com/wsoll/articles/blob/main/iterables-memory-tracing.md)
2. [Behaviour of processes launched with Python](https://github.com/wsoll/articles/blob/main/launching-processes.md)
3. [Behaviour of processes launched with Python - part 2](https://github.com/wsoll/articles/blob/main/launching-processes-2.md)
4. [Diamond inheritance problem](https://github.com/wsoll/articles/blob/main/diamond-inheritance-problem.md)

---

```python
import asyncio
import multiprocessing
import sys
import time
from asyncio import Future
from concurrent.futures import ProcessPoolExecutor
from concurrent.futures.process import BrokenProcessPool
from typing import Literal, Callable
from mem_trace import print

WORKERS = 8
WORKER_EXECUTION_TIME_SEC = 5


def process():
    print("Hello process! Executing...")
    time.sleep(WORKER_EXECUTION_TIME_SEC)
    print("Done.")
    return 0


def evil_dream():
    try:
        print(
            "I welcome you, Python, my old friend. But to this place where destiny is made, why have you come "
            "unprepared?"
        )
        evil_dream()
    except:
        print(
            "What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How "
            "could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay "
            "down your weapons. It is not too late for my mercy."
        )
        evil_dream()


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
        except BrokenProcessPool as e:
            print(e)
            raise e


async def main():
    print(f"Python version: {sys.version}")
    start_process_method = sys.argv[1]
    if start_process_method not in ("spawn", "fork"):
        raise RuntimeError("available arguments are fork and spawn.")
    process_pool = ProcessPool(WORKERS, start_process_method)

    workers_tasks = [asyncio.create_task(process_pool.run_worker(process)) for _ in range(WORKERS - 1)]
    workers_tasks.append(asyncio.create_task(process_pool.run_worker(evil_dream)))

    for task in workers_tasks:
        try:
            await asyncio.wait_for(task, timeout=WORKER_EXECUTION_TIME_SEC + 1)
        except asyncio.TimeoutError:
            print("STOP IT NOW!")
            task.cancel()
        except BrokenProcessPool as e:
            print(e)


if __name__ == "__main__":
    asyncio.run(main())

```

- Python 3.9.6
- Darwin Kernel Version 23.4.0
```text
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
```text
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
  SpawnProcess-1 [ 5.00s] Done.
  SpawnProcess-5 [ 5.00s] Done.
  SpawnProcess-7 [ 5.01s] Done.
  SpawnProcess-6 [ 5.00s] Done.
  SpawnProcess-2 [ 5.00s] Done.
  SpawnProcess-8 [ 5.00s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [ 5.00s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
  SpawnProcess-3 [ 5.01s] Done.
  SpawnProcess-4 [ 5.00s] Done.
  ...
  SpawnProcess-8 [10.99s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [10.99s] I welcome you, Python, my old friend. But to this place where destiny is made, why have you come unprepared?
MainProcess      [11.20s] STOP IT NOW!
  SpawnProcess-8 [11.00s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  SpawnProcess-8 [11.01s] What a fool you are. I'm a god. How can you kill a god? What a grand and intoxicating innocence. How could you be so naive? There is no escape. No Recall or Intervention can work in this place. Come. Lay down your weapons. It is not too late for my mercy.
  ...
```
