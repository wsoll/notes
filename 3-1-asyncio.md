# Event loop

```python
import asyncio


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.call_later(2, loop.stop)
    loop.call_soon(print, "Hello!")
    loop.run_forever()
    loop.close()

```

# time.sleep() vs asyncio.sleep()
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
2. The loop should be stopped within 15 seconds, however the event loop is blocked for the sleep time.

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


if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    loop.call_later(10, loop.stop)
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