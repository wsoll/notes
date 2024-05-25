- [Global Interpreter Lock (GIL) and Concurrency](#global-interpreter-lock--gil--and-concurrency)
- [Asyncio goal](#asyncio-goal)
  * [time.sleep vs asyncio.sleep](#timesleep-vs-asynciosleep)
  * [asyncio.run vs loop.run_until_complete](#asynciorun-vs-looprun_until_complete)
  * [task, gather, wait](#task-gather-wait)
  * [lock, queue](#lock-queue)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Global Interpreter Lock (GIL) and Concurrency
Global interpreter lock causes to use single thread at the time. It means there is one lock per process. The reason to
apply GIL was to protect the internal state of the interpreted from being corrupted if multiple threads would start
to change it.

However, making code faster by disabling GIL and applying multiply locks in specific places to speed up the code is not
that easy task and many times fails, therefore GIL is still present.

# Asyncio goal
is to maximize the usagew of a single thread by handling I/O asynchronously by enabling concurrency using coroutines:


```python
import asyncio


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.call_later(2, loop.stop)
    loop.call_soon(print, "Hello!")
    loop.run_forever()
    loop.close()

```

## time.sleep vs asyncio.sleep
```python
import asyncio
import multiprocessing
import time
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


def sleeping_func(i):
    print("Hello, sleeping_func!")
    time.sleep(i)
    print("Done.")


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.call_later(15, loop.stop)
    for i in range(1, 10):
        loop.call_soon(sleeping_func, i)

    try:
        loop.run_forever()
    finally:
        loop.close()


```
Result:
```commandline
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 1.01s] Done.
MainProcess      [ 1.01s] Hello, sleeping_func!
MainProcess      [ 3.01s] Done.
MainProcess      [ 3.01s] Hello, sleeping_func!
MainProcess      [ 6.02s] Done.
MainProcess      [ 6.02s] Hello, sleeping_func!
MainProcess      [10.03s] Done.
MainProcess      [10.03s] Hello, sleeping_func!
MainProcess      [15.03s] Done.
MainProcess      [15.03s] Hello, sleeping_func!
MainProcess      [21.04s] Done.
MainProcess      [21.04s] Hello, sleeping_func!
MainProcess      [28.04s] Done.
MainProcess      [28.04s] Hello, sleeping_func!
MainProcess      [36.05s] Done.
MainProcess      [36.05s] Hello, sleeping_func!
MainProcess      [45.05s] Done.
```

1. time.sleep() is a synchronous function that blocks event loop. As a result every call is sequential.
2. The loop should be stopped within 15 seconds (call later -> loop.stop), however the event loop is blocked for the sleep time.

To resolve the issue dedicated asynchronous sleep method that respects event loop has to be applied:

```python
import asyncio
import multiprocessing
import time
from rich.console import Console

...

async def sleeping_func(i):
    print("Hello, sleeping_func!")
    await asyncio.sleep(i)
    print("Done.")


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.call_later(15, loop.stop)
    for i in range(1, 10):
        loop.create_task(sleeping_func(i))

    try:
        loop.run_forever()
    finally:
        loop.close()

```

```commandline
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 1.00s] Done.
MainProcess      [ 2.00s] Done.
MainProcess      [ 3.00s] Done.
MainProcess      [ 4.00s] Done.
MainProcess      [ 5.00s] Done.
MainProcess      [ 6.00s] Done.
MainProcess      [ 7.00s] Done.
MainProcess      [ 8.00s] Done.
MainProcess      [ 9.00s] Done.
```

- dedicated asyncio.sleep() should be used with 'await' keyword
- 'await' keyword can be used in coroutine (asynchronous function -> async def foo(): ...)

## asyncio.run vs loop.run_until_complete
To simply run some asynchronous code you could use:
```python
import asyncio

async def foo():
    ...


asyncio.run(foo())
```

However, if you need finer control over the event loop such as embedding it in a large app, running multiple coroutines 
and have control over it:


```python
import asyncio

async def main():
    ...

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    try:
        loop.run_until_complete(main())
    finally:
        loop.close()

```


## task, gather, wait
Awaiting on a coroutine actually doesn't schedule it for concurrent execution. To do so, you have to create a task:

```python
import asyncio
import multiprocessing
import time
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


async def nested_coroutine(i):
    print("Hello, nested coroutine!")
    await asyncio.sleep(i)
    print("nested coroutine is finished.")
    return i


async def sleeping_func(i):
    print("Hello, sleeping_func!")
    result = await nested_coroutine(i)
    print("sleeping_func is finished.")
    return result


async def main():
    task_worker = asyncio.create_task(sleeping_func(1))
    # Some concurrent code
    await task_worker
    print(task_worker.result())


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    try:
        loop.run_until_complete(main())
    finally:
        loop.close()

```
```commandline
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 1.00s] nested coroutine is finished.
MainProcess      [ 1.01s] sleeping_func is finished.
MainProcess      [ 1.01s] 1
```


You could 'gather' tasks if you require running multiple asynchronous tasks and to collect their result in a single 
future:
```python
...

async def main():
    tasks_workers = [asyncio.create_task(sleeping_func(i)) for i in range(4)]
    # Some concurrent code
    await asyncio.gather(*tasks_workers)
    for task_worker in tasks_workers:
        print(task_worker.result())

...

```
```commandline
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] nested coroutine is finished.
MainProcess      [ 0.00s] sleeping_func is finished.
MainProcess      [ 1.00s] nested coroutine is finished.
MainProcess      [ 1.01s] sleeping_func is finished.
MainProcess      [ 2.00s] nested coroutine is finished.
MainProcess      [ 2.01s] sleeping_func is finished.
MainProcess      [ 3.00s] nested coroutine is finished.
MainProcess      [ 3.01s] sleeping_func is finished.
MainProcess      [ 3.01s] 0
MainProcess      [ 3.01s] 1
MainProcess      [ 3.01s] 2
MainProcess      [ 3.01s] 3
```


In case of control over the awaiting behaviour you could use wait method for timeouts or different completion 
conditions:
```python
...

async def main():
    tasks_workers = [asyncio.create_task(sleeping_func(i)) for i in range(4)]
    # Some concurrent code

    done, pending = await asyncio.wait(tasks_workers, timeout=2)

    for task in done:
        print(f"{task.get_name()} is finished. Result: {task.result()}")

    for task in pending:
        print(f"{task.get_name()} is still pending...Cancelling...")
        task.cancel()
...

```

```commandline
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] Hello, sleeping_func!
MainProcess      [ 0.00s] Hello, nested coroutine!
MainProcess      [ 0.00s] nested coroutine is finished.
MainProcess      [ 0.00s] sleeping_func is finished.
MainProcess      [ 1.00s] nested coroutine is finished.
MainProcess      [ 1.01s] sleeping_func is finished.
MainProcess      [ 2.00s] Task-3 is finished. Result: 1
MainProcess      [ 2.00s] Task-2 is finished. Result: 0
MainProcess      [ 2.01s] Task-5 is still pending...Cancelling...
MainProcess      [ 2.01s] Task-4 is still pending...Cancelling...
```

## lock, queue

If you need to control access to a shared resources like: lists, dicts, or any mutable objects, file or network 
connection, a lock can ensure that only one coroutine uses the resource at any given time, preventing conflicts or
corruption:

```python
...

async def my_worker(lock):
    print("Attempting to attain lock")

    async with lock:
        print("Currently locked")
        await asyncio.sleep(2)

    print("Unlocked from critical section")


async def main():
    lock = asyncio.Lock()
    tasks = [asyncio.create_task(my_worker(lock)) for i in range(2)]

    await asyncio.wait(tasks)


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
    print("All workers completed.")
    loop.close()

```

```commandline
MainProcess      [ 0.00s] Attempting to attain lock
MainProcess      [ 0.00s] Currently locked
MainProcess      [ 0.00s] Attempting to attain lock
MainProcess      [ 0.00s] Currently locked
MainProcess      [ 2.00s] Unlocked from critical section
MainProcess      [ 2.01s] Unlocked from critical section
MainProcess      [ 2.01s] All workers completed.
```

Do not use the lock in case of non-shared resources, read-only access (not needed) or high-performance scenario (slows
it down).

To coordinate work between coroutines like producer & consumer scenario a non-thread safe but native for asyncio queue 
is recommended to use:

```python
...

async def producer(my_queue: asyncio.Queue):
  while True:
    await asyncio.sleep(1)
    print("Putting new item onto queue.")
    await my_queue.put(random.randint(1, 5))


async def consumer(id: int, my_queue: asyncio.Queue):
  while True:
    print(f"Consumer: {id} attempting to get from queue.")
    item = await my_queue.get()
    print(f"Consumer: {id} consumed item with id: {item}")


async def main():
  queue = asyncio.Queue(maxsize=10)
  producers = [asyncio.create_task(producer(queue)) for _ in range(2)]
  consumers = [asyncio.create_task(consumer(i, queue)) for i in range (1,3)]
  tasks = producers + consumers
  await asyncio.wait(tasks, timeout=5)

  for task in tasks:
    task.cancel()

if __name__ == "__main__":
  loop = asyncio.get_event_loop()
  loop.run_until_complete(main())
  loop.close()


```

```commandline
MainProcess      [ 0.00s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 0.00s] Consumer: 2 attempting to item get from queue. Waiting...
MainProcess      [ 1.00s] Putting new item onto queue.
MainProcess      [ 1.00s] Putting new item onto queue.
MainProcess      [ 1.00s] Consumer: 1 consumed item with id: 3
MainProcess      [ 1.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 1.01s] Consumer: 1 consumed item with id: 2
MainProcess      [ 1.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 2.00s] Putting new item onto queue.
MainProcess      [ 2.01s] Putting new item onto queue.
MainProcess      [ 2.01s] Consumer: 1 consumed item with id: 5
MainProcess      [ 2.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 2.01s] Consumer: 1 consumed item with id: 5
MainProcess      [ 2.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 3.01s] Putting new item onto queue.
MainProcess      [ 3.01s] Putting new item onto queue.
MainProcess      [ 3.01s] Consumer: 1 consumed item with id: 5
MainProcess      [ 3.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 3.01s] Consumer: 1 consumed item with id: 1
MainProcess      [ 3.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 4.01s] Putting new item onto queue.
MainProcess      [ 4.01s] Consumer: 1 consumed item with id: 1
MainProcess      [ 4.01s] Consumer: 1 attempting to item get from queue. Waiting...
MainProcess      [ 4.01s] Putting new item onto queue.
MainProcess      [ 4.02s] Consumer: 2 consumed item with id: 2
MainProcess      [ 4.02s] Consumer: 2 attempting to item get from queue. Waiting...
```