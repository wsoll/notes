# Reasons to apply

Existence of Global Interpreter Lock (GIL) - the reason to apply GIL was to protect the internal state of the interpreter from being corrupted if multiple thread would change it.
One of potential issues with it could be a CPU leak that affects other tasks.
<!--- TODO: example -->

Asyncio allows to maximize the usage of a single thread by:
- Handling I/O **asynchroniously**.
- Enabling concurrent code using **coroutines**.

If it's needed to use multiple threads -> use multiple coroutines with multiple Python processes.

## Data Sink

Let's generate some data
```python
import datetime
import pickle
import random
import sys
import tracemalloc

import numpy as np
import pandas as pd

from mem_trace import print, mem_usage

RECORDS_COUNT = 100_000_000


def generate_data(length: int) -> np.recarray:
    print("Generating data to list.", mem_usage())
    values = [0, 1, np.nan]
    data = [(datetime.datetime.now(), random.choice(values)) for _ in range(length)]
    print("Converting list into data frame.", mem_usage())
    df = pd.DataFrame(data, columns=['timestamp', 'val'])
    np_array = df.to_records()
    return np_array


if __name__ == "__main__":
    tracemalloc.start()
    print(f"Generating {RECORDS_COUNT:_} records of data...")
    data = generate_data(RECORDS_COUNT)
    print(f"Generated {sys.getsizeof(data) / (1024 * 1024):.2f} MB of data. Pickling.", mem_usage())
    with open('data.pickle', 'wb') as f:
        pickle.dump(data, f)
    print("Done.")

```

```text
MainProcess      [ 0.00s] Generating 100_000_000 records of data...
MainProcess      [ 0.01s] Generating data to list. Memory usage: 0 MB; peak: 0 MB
MainProcess      [145.60s] Converting list into data frame. Memory usage: 9951 MB; peak: 9951 MB
MainProcess      [182.08s] Generated 2288.82 MB of data. Pickling. Memory usage: 2288 MB; peak: 16246 MB
MainProcess      [ 1.56s] Done.
```
