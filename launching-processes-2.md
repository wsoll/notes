1. [Lazy loading and collection generation peak](https://github.com/wsoll/articles/blob/main/iterables-memory-tracing.md)
2. [Behaviour of processes launched with Python](https://github.com/wsoll/articles/blob/main/launching-processes.md)
3. [Behaviour of processes launched with Python - part 2](https://github.com/wsoll/articles/blob/main/launching-processes-2.md)
4. [Diamond inheritance problem](https://github.com/wsoll/articles/blob/main/diamond-inheritance-problem.md)

---

```python
import asyncio
import multiprocessing
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
    process_pool = ProcessPool(WORKERS, "spawn")

    workers_tasks = [asyncio.create_task(process_pool.run_worker(process)) for _ in range(WORKERS - 1)]
    workers_tasks.append(asyncio.create_task(process_pool.run_worker(evil_dream)))

    for task in workers_tasks:
        try:
            await asyncio.wait_for(task, timeout=6)
        except asyncio.TimeoutError:
            task.cancel()
        except BrokenProcessPool as e:
            print(e)


if __name__ == "__main__":
    asyncio.run(main())


```
