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
from typing import Literal, Callable
from mem_trace import print

WORKERS = 8
WORKER_EXECUTION_TIME_SEC = 60


def process():
    time.sleep(WORKERS)
    print("Hello process!")
    return 0


def evil_process():
    try:
        evil_process()
    except:
        evil_process()


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
        return await loop.run_in_executor(self.executor, process_function)


async def main():
    process_pool = ProcessPool(WORKERS, "spawn")
    worker_tasks = [asyncio.create_task(process_pool.run_worker(process)) for _ in range(WORKERS - 1)]
    worker_tasks.append(asyncio.create_task(process_pool.run_worker(evil_process)))

    try:
        await asyncio.gather(*worker_tasks)
    except Exception as e:
        print(f"{type(e)}: {e}")


if __name__ == "__main__":
    asyncio.run(main())

```
