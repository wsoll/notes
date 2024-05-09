```python
import asyncio
import multiprocessing
import time
import tracemalloc
from asyncio import Future
from concurrent.futures import ProcessPoolExecutor
from typing import Literal, Callable
from mem_trace import print, mem_usage

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
        self.__executor = ProcessPoolExecutor(
            max_workers=workers_count, mp_context=context, initializer=tracemalloc.start
        )
        self.__running_workers = []
        self.__finished_workers = []

    async def run_worker(self, process_function: Callable) -> Future:
        loop = asyncio.get_running_loop()
        return loop.run_in_executor(self.__executor, process_function)


async def main():
    process_pool = ProcessPool(WORKERS, "fork")
    worker_tasks = [asyncio.create_task(process_pool.run_worker(process)) for _ in range(WORKERS - 1)]
    worker_tasks.append(asyncio.create_task(process_pool.run_worker(evil_process)))

    await asyncio.gather(*worker_tasks)


if __name__ == "__main__":
    asyncio.run(main())

```
