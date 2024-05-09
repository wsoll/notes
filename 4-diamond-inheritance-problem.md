1. [Lazy loading and collection generation peak](https://github.com/wsoll/articles/blob/main/iterables-memory-tracing.md)
2. [Behaviour of processes launched with Python](https://github.com/wsoll/articles/blob/main/launching-processes.md)
3. [Behaviour of processes launched with Python - part 2](https://github.com/wsoll/articles/blob/main/launching-processes-2.md)
4. [Diamond inheritance problem](https://github.com/wsoll/articles/blob/main/diamond-inheritance-problem.md)

---

Programming languages that allow only single inheritance don't have the problem. Anyway Python allows multiple inheritance...:

```python
class A:
    def __init__(self):
        print("A")


class B(A):
    def __init__(self):
        print("B")
        super().__init__()


class C(A):
    def __init__(self):
        print("C")
        super().__init__()


class D(B, C):
    def __init__(self):
        print("D")
        super().__init__()


if __name__ == "__main__":
    d = D()
```

```text
D
B
C
A
```

1. It is recommended to use super() builtin to do not encounter the issue or duplicate initialization (for initializing parents manually).
2. Order of inheritance for subclass defines initialization order. 
