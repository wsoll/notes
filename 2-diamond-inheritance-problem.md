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

```commandline
D
B
C
A
```

1. It is recommended to use super() builtin to do not encounter the issue or duplicate initialization (for initializing parents manually).
2. Order of inheritance for subclass defines initialization order. 
