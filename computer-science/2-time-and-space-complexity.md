- [Definition](#definition)
- [Time complexity types](#time-complexity-types)
  * [Constant](#constant)
  * [Linear](#linear)
  * [Quadratic](#quadratic)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Definition
**Time complexity** - a way of showing how the runtime of a function increases as the size of the input increases

time complexity types:

- constant: O(1)
- linear: O(n)
- quadratic: O(n<sup>2</sup>)
- logarithmic: O(log n)
- cubic: O(n<sup>3</sup>)
- exponential: O(2<sup>n</sup>)
- factorial: O(n!)

**How to count time complexity?**
1.  write down the function formally
2.  find the fastest growing term
3.  take out the coefficient

**In computer science you tend to care more about larger input than smaller input.**

# Time complexity types

## Constant

```python
arr = [1, 4, 3, 2, ..., 10]

def stupid_function(arr):
	total = 0  # O(1)
	return total  # O(1)
```

Theoretically

T = O(1) + O(1) = c<sub>1</sub> + c<sub>2</sub> = c<sub>3</sub> = c<sub>3</sub> * 1 = O(1)

O(1) + O(1) = O(1)

* O(1) + O(1) - There are two constant time operations
* c<sub>1</sub> + c<sub>2</sub> - Actual constant time for these operations
* c<sub>3</sub> - By adding these two constants we have another constant.
* c<sub>3</sub> * 1 - representation which still denotes a constant time operation
* O(1) + O(1) - shows that the sum fo two constant time operations is still a constant
time operation.

By an experiment

T = c = 0.115 * 1 = ~~c<sub>1</sub>~~ * 1 = O(1)
* 0.115 - represents a specific constant time taken for an operation.
* \* 1 - emphasizes that it is a single operation
* ~~c<sub>1</sub>~~ * 1 - Crossed constant variable emphasizes that time complexity 
depends on number of operations
* O(1) + O(1) = O(1) - Shows that the sum of two constant time operations is still a
constant time operation. Sum of two constants is still constant.

## Linear

```python
given_array = [1, 4, 3, 2, ..., 10]

def find_sum(given_array):
    total = 0  # O(1)
    for i in given_array:  # O(n)
        total += 1  # O(1)
    return total  # O(1)
```

Theoretically

T = O(1) + n * O(1) + O(1) = ~~c<sub>1</sub>~~ + n * ~~c<sub>2</sub>~~ = O(n)
* O(1) + ... + O(1) - represents constant operation for first and third term (as for 
loop with its body is considered as another term)
* O(n) * 1 - a linear number (n) of constant operations
* ~~c<sub>1</sub>~~ + n * ~~c<sub>2</sub>~~ - simplified first and third term within
c<sub>1</sub> as adding these gives constant time anyway. The loop term is constant as 
well, and it depends on size of the array.
* O(n) - Indicates the time complexity of the algorithm (**the fastest growing term**)

By the experiment

T = an + b = ~~c<sub>1</sub>~~ * **n** + ~~c<sub>2</sub>~~ = O(n)
* an - represents the loop part, where an is the body (constant term) and n the loop 
(size of the array)
* b - represents other constant terms
* ~~c<sub>1</sub>~~ * **n** + ~~c<sub>2</sub>~~ - by simplifying the constants as these 
don't depend on the size of the input, so we can drop them, and we're left with n only.
* O(n) - as a result O(n) indicates the time complexity (**the fastest growing term**)

## Quadratic

```python
array_2d = [
	[1, 4, 3],
	[5, 3, 2],
	[6, 1, 2]
]  # n^2

def find_sum_2d(array_2d):
	total = 0  # O(1)
	for row in array_2d:  # O(n)
		for i in row:  # O(n)
			total += i  # O(1)
	return total  # O(1)
```

Theoretically:

T = O(1) + n<sup>2</sup> * (n...O(1)) + O(1) = ~~c<sub>1</sub>~~ + n<sup>2</sup> * ~~c<sub>2</sub>~~ = O(n<sup>2</sup>)
* O(1) + ... + O(1) - represents constant operation for first and third term (as for
  loop with its body is considered as another term)
* n<sup>2</sup> - represents the outer loop. Iterates over each row in array_2d,
running for each row, which results in n. Since this loop is nested inside another loop
that runs n times, it effectively runs n<sup>2</sup> times in total
* (n...O(1)) - represents the inner loop. Iterates over each element in the row and adds 
to total.
* ~~c<sub>1</sub>~~ + n<sup>2</sup> * ~~c<sub>2</sub>~~- by simplifying the constants as 
they don't depend on the size of the input, we can drop them. The fastest growing term
is the middle one, so quadratic with respect to the size of the input.

By the experiment

T = cn<sup>2</sup> + dn + e = ~~c<sub>1</sub>~~ * n<sup>2</sup> + ~~c<sub>2</sub>~~ * n + ~~c<sub>3</sub>~~ = **n<sup>2</sup>** + **n** = O(n<sup>2</sup>)

* Similarly to previous examples, the fastest growing term is in outerloop, therefore n as well as constants could be 
dropped.
 

**What if there is an execution of two double loops in a function?**

```python
def find_sum_2d(array_2d):
    total = 0  # O(1)
    for row in array_2d:  # O(n)
        for i in row:  # O(n)
            total += i  # O(1)

    for row in array_2d:  # O(n)
        for i in row:  # O(n)
            total += i  # O(1)
            
    return total  # O(1)
```

T = O(2n<sup>2</sup>) = O(n<sup>2</sup>)

Q: why?

A: In Big O notation, constants are ignored. So, when we say O(2n2)O(2n2), we can drop the 
constant factor 2 because it doesn't significantly affect the growth rate of the 
function as nn approaches infinity.

T = 2n<sup>2</sup> * c + ... = 2n<sup>2</sup> * c + c<sub>2</sub>n + c<sub>3</sub>a = (2c) * n<sup>2</sup> + c<sub>2</sub>n + 3 = O(n<sup>2</sup>)