
# Definition
**Time complexity** - a way of showing how the runtime of a function increases as the size of the input increases

time complexity types:

- constant time: O(1)
- linear time: O(n)
- quadratic time: O(n<sup>2</sup>)

**How to count time complexity?**
1.  write down the function formally
2.  find the fastest growing term
3.  take out the coefficient

**In computer science you tend to care more about larger input than smaller input.**

# Experiments

T - time complexity
c<sub>x</sub> - constant

## Constant

```python
given_array = [1, 4, 3, 2, ..., 10]

def stupid_function(given_array):
	total = 0  # O(1)
	return total  # O(1)
```

Theoretically:
T = O(1) + O(1) = c<sub>1</sub> + c<sub>2</sub> = c<sub>3</sub> = c<sub>3</sub> * 1 = O(1)
O(1) + O(1) = O(1)

By the experiment:
T = c = 0.115 * 1 = ~~c<sub>1</sub>~~ * 1 = O(1)

## Linear

```python
given_array = [1, 4, 3, 2, ..., 10]

def find_sum(given_array):
    total = 0  # O(1)
    for i in given_array:  # O(1)
        total += 1  # O(1)
    return total  # O(1)
```

Theoretically:
T = O(1) + n * O(1) + O(1) = ~~c<sub>1</sub>~~ + n * ~~c<sub>2</sub>~~ = O(n)

By the experiment
T = an + b = ~~c<sub>1</sub>~~ * **n** + ~~c<sub>2</sub>~~ = O(n)

## Quadratic

```python
array_2d = [
	[1, 4, 3],
	[5, 3, 2],
	[6, 1, 2]
]  # n^2

def find_sum_2d(array_2d):
	total = 0  # O(1)
	for row in array_2d:  # O(1)
		for i in row:  # O(1)
			total += i  # O(1)
	return total  # O(1)
```

Theoretically:
T = O(1) + n<sup>2</sup> * (n...O(1)) + O(1) = ~~c<sub>1</sub>~~ + n<sup>2</sup> * ~~c<sub>2</sub>~~ = O(n<sup>2</sup>)

T = cn<sup>2</sup> + dn + e = ~~c<sub>1</sub>~~ \* n<sup>2</sup> + ~~c<sub>2</sub>~~ \* n + ~~c<sub>3</sub>~~ = **n<sup>2</sup>** + **n** = O(n<sup>2</sup>)

**What if there is an execution of two double loops in a function?**
T = O(2n<sup>2</sup>) = O(n<sup>2</sup>)
why?
T = 2n<sup>2</sup> * c + ... = 2n<sup>2</sup> * c + c<sub>2</sub>n + c<sub>3</sub>a = (2c) * n<sup>2</sup> + c<sub>2</sub>n + 3 = O(n<sup>2</sup>)