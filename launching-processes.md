
# Start Process Method
<table><tr><th></th>
	<th scope="col"al>Fork</th>
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

## Example 1 - Module scope mutable variable

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

### Fork
```text
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

### Spawn
```text
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

## Example 2 - Workers logger

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
### Spawn
```text
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4390984784): start subprocess with 'spawn' method
SpawnProcess-2 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
SpawnProcess-1 - __mp_main__ - Print[MODULE SCOPE]: module scoped reached.
```
- Missing logs for call_log function call

#### Modification 1 - pass init_log function as ProcessPoolExecutor initializer
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

### Fork
```text
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4354415696): start subprocess with 'fork' method
ForkProcess-1 - __main__:   Logger(id: 4354415696): function executed.
ForkProcess-2 - __main__:   Logger(id: 4354415696): function executed.
```
- Same logger_ids

#### Modification 1 - move call_log to separate.py module, initialize own logger on module scope

```text
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4360363344): start subprocess with 'fork' method
ForkProcess-1 - separate:   Logger(id: 4367811088): function executed.
ForkProcess-2 - separate:   Logger(id: 4367811088): function executed.
```
- Different logger ids for main and forked processes

#### Modification 2 - pass init_log function as ProcessPoolExecutor initializer

```text
MainProcess - __main__ - Print[MODULE SCOPE]: module scoped reached.
MainProcess - __main__:   Logger(id: 4325820432): start subprocess with 'fork' method
ForkProcess-1 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-1 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-2 - separate:   Logger(id: 4327030224): function executed.
ForkProcess-2 - separate:   Logger(id: 4327030224): function executed.
```
- Logs duplication for workers. Contains two handlers - first from parent, second for worker initalizer
