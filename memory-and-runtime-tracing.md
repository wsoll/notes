# Memory tracing
```python
import multiprocessing
import time
import tracemalloc

from rich.console import Console

console = Console()
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
    print_ts = now


def mem_usage() -> str:
    current, peak = tracemalloc.get_traced_memory()
    return f"Memory usage: {current//1024//1024} MB; peak: {peak//1024//1024} MB"

```

## List vs Generator

```python
from memory_check import print, mem_usage

n = 20000000

generator_argument_result = sum(x ** 2 for x in range(n))
print("Generator argument", mem_usage())
collection_argument_result = sum([x ** 2 for x in range(n)])
print("List argument", mem_usage())

```
Result
```text
MainProcess      [11.34s] Generator argument Memory usage: 7 MB; peak: 7 MB
MainProcess      [13.85s] List argument Memory usage: 7 MB; peak: 771 MB
```
