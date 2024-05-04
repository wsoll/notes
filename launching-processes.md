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
import os
import sys
from concurrent.futures import ProcessPoolExecutor
from time import sleep

MODULE_SCOPE_LIST = ["Initial value"]
print(f"(1) Module scope reached, pid: {os.getpid()}")


def append_to_shared_list(count):
    sleep(count)
    print(
        f"(5) List before subprocess {count} scope update ({count} sec sleep): "
        f"{MODULE_SCOPE_LIST}, list_id: {id(MODULE_SCOPE_LIST)}, pid: {os.getpid()}"
    )
    MODULE_SCOPE_LIST.append(f"Child process {count} update")
    print(
        f"(6) List after subprocess {count} update ({count} sec sleep): "
        f"{MODULE_SCOPE_LIST}, list_id: {id(MODULE_SCOPE_LIST)}, pid: {os.getpid()}"
    )


async def main(start_process_method):
    MODULE_SCOPE_LIST.append(f"Parent process update")
    print(
        f"(4) List after process scope update: "
        f"{MODULE_SCOPE_LIST}, list_id: {id(MODULE_SCOPE_LIST)}, pid: {os.getpid()}"
    )

    context = multiprocessing.get_context(start_process_method)
    with ProcessPoolExecutor(max_workers=2, mp_context=context) as executor:
        executor.map(append_to_shared_list, range(1, 3))
    print(f"(7) List after all updates: {MODULE_SCOPE_LIST}, list_id: {id(MODULE_SCOPE_LIST)}, pid: {os.getpid()}")


if __name__ == "__main__":
    start_process_method = sys.argv[1]
    print(f"(2) Initial list value: {MODULE_SCOPE_LIST}, id:{id(MODULE_SCOPE_LIST)}, pid: {os.getpid()}")
    print(f"(3) Start subprocess with '{start_process_method}' method")
    asyncio.run(main(start_process_method))
    print()

```

### Fork
```text
(1) Module scope reached, pid: 7199
(2) Initial list value: ['Initial value'], list_id:4317213696, pid: 7199
(3) Start subprocess with 'fork' method
(4) List after process scope update: ['Initial value', 'Parent process update'], list_id: 4317213696, pid: 7199
(5) List before subprocess 1 scope update (1 sec sleep): ['Initial value', 'Parent process update'], list_id: 4317213696, pid: 7200
(6) List after subprocess 1 update (1 sec sleep): ['Initial value', 'Parent process update', 'Child process 1 update'], list_id: 4317213696, pid: 7200
(5) List before subprocess 2 scope update (2 sec sleep): ['Initial value', 'Parent process update'], list_id: 4317213696, pid: 7201
(6) List after subprocess 2 update (2 sec sleep): ['Initial value', 'Parent process update', 'Child process 2 update'], list_id: 4317213696, pid: 7201
(7) List after all updates: ['Initial value', 'Parent process update'], list_id: 4317213696, pid: 7199
```
**Consequence of the same memory space & copy-on-write resource utilization:**

- (2), (4) list_id is the same than (5), (6) for pid 7200 & pid 7201)
-  Every subprocess contains 'Parent process update'

### Spawn
```text
(1) Module scope reached, pid: 7202
(2) Initial list value: ['Initial value'], list_id:4326470656, pid: 7202
(3) Start subprocess with 'spawn' method
(4) List after process scope update: ['Initial value', 'Parent process update'], list_id: 4326470656, pid: 7202
(1) Module scope reached, pid: 7204
(1) Module scope reached, pid: 7205
(5) List before subprocess 1 scope update (1 sec sleep): ['Initial value'], list_id: 4346200000, pid: 7204
(6) List after subprocess 1 update (1 sec sleep): ['Initial value', 'Child process 1 update'], list_id: 4346200000, pid: 7204
(5) List before subprocess 2 scope update (2 sec sleep): ['Initial value'], list_id: 4350476224, pid: 7205
(6) List after subprocess 2 update (2 sec sleep): ['Initial value', 'Child process 2 update'], list_id: 4350476224, pid: 7205
(7) List after all updates: ['Initial value', 'Parent process update'], list_id: 4326470656, pid: 7202
```

**Consequence of new process launching with a new executable file load into memory & initializing a new runtime environement:**

- (2), (4) list_id is different than (5), (6) for pid 7204 & 7205
- (1) printed three times (instruction on module scope)
- Any subprocess doesn't contain 'Parent process update' made in function scope (no read for same memory space).


## Example 2 - Workers logger

```python 
import asyncio
import logging
import multiprocessing
import os
import sys
from concurrent.futures import ProcessPoolExecutor
from time import sleep


print(f"(1) Module scope reached, pid: {os.getpid()}")

logger = logging.getLogger(__name__)


def call_logger(count):
    sleep(count)
    logger.debug(f"Some_func executed in subprocess {count}, pid: {os.getpid()}, logger_id: {id(logger)}")


def init_logger():
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)

    handler = logging.StreamHandler(sys.stdout)
    handler.setLevel(logging.DEBUG)
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    handler.setFormatter(formatter)
    root.addHandler(handler)


async def main(start_process_method):
    context = multiprocessing.get_context(start_process_method)
    with ProcessPoolExecutor(max_workers=2, mp_context=context) as executor:
        executor.map(call_logger, range(1, 3))


if __name__ == "__main__":
    # TODO: Show 'copy-on-write' for 'fork' subprocess method.
    init_logger()
    start_process_method = sys.argv[1]
    logger.debug(f"(2) Start subprocess with '{start_process_method}' method, logger_id {id(logger)}")
    asyncio.run(main(start_process_method))
```

### Fork
```text
(1) Module scope reached, pid: 8420
2024-04-29 17:04:18,421 - __main__ - DEBUG - (2) Start subprocess with 'fork' method, logger_id 4385712080
2024-04-29 17:04:19,427 - __main__ - DEBUG - Some_func executed in subprocess 1, pid: 8421, logger_id: 4385712080
2024-04-29 17:04:20,427 - __main__ - DEBUG - Some_func executed in subprocess 2, pid: 8422, logger_id: 4385712080
```
- Same logger_ids

#### Modification 1 - move call_log to separate module, initialize own logger on module scope

```bash
(1) Module scope reached, pid: 8465
2024-04-29 17:08:29,181 - __main__ - DEBUG - (2) Start subprocess with 'fork' method, logger_id 4373173456
2024-04-29 17:08:30,186 - bar - DEBUG - Some_func executed in subprocess 1, pid: 8466, logger_id: 4388064784
2024-04-29 17:08:31,186 - bar - DEBUG - Some_func executed in subprocess 2, pid: 8467, logger_id: 4388064784
```
- Different logger children (\__main__ and bar) with different logger_ids

#### Modification 2 - pass init_log function as ProcessPoolExecutor initializer

```text
(1) Module scope reached, pid: 8508
2024-04-29 17:13:24,931 - __main__ - DEBUG - (2) Start subprocess with 'fork' method, logger_id 4371598288
2024-04-29 17:13:25,935 - bar - DEBUG - Some_func executed in subprocess 1, pid: 8509, logger_id: 4379299152
2024-04-29 17:13:25,935 - bar - DEBUG - Some_func executed in subprocess 1, pid: 8509, logger_id: 4379299152
2024-04-29 17:13:26,935 - bar - DEBUG - Some_func executed in subprocess 2, pid: 8510, logger_id: 4379299152
2024-04-29 17:13:26,935 - bar - DEBUG - Some_func executed in subprocess 2, pid: 8510, logger_id: 4379299152
```
- Logs duplication for workers. Contains two handlers - first from parent, second for worker initalizer

### Spawn
```text
(1) Module scope reached, pid: 8534
2024-04-29 17:16:42,342 - __main__ - DEBUG - (2) Start subprocess with 'spawn' method, logger_id 4394772432
(1) Module scope reached, pid: 8539
(1) Module scope reached, pid: 8540
```
- Missing logs for call_log function call

#### Modification 1 - pass init_log function as ProcessPoolExecutor initializer
```text
(1) Module scope reached, pid: 8572
2024-04-29 17:19:15,152 - __main__ - DEBUG - (2) Start subprocess with 'spawn' method, logger_id 4385416976
(1) Module scope reached, pid: 8574
(1) Module scope reached, pid: 8575
2024-04-29 17:19:16,209 - bar - DEBUG - Some_func executed in subprocess 1, pid: 8574, logger_id: 4389380880
2024-04-29 17:19:17,209 - bar - DEBUG - Some_func executed in subprocess 2, pid: 8575, logger_id: 4315243536
```
- All logs visible
- Separate logger_ids
