````md
# Bitwise Operators Interview Questions

## Beginner Level

### 1. What are bitwise operators?
- Explain what bitwise operators are.
- Why are they faster than some arithmetic operations?

### 2. List all bitwise operators in C/C++/Java.
- `&` (AND)
- `|` (OR)
- `^` (XOR)
- `~` (NOT)
- `<<` (Left Shift)
- `>>` (Right Shift)

---

### 3. What is the difference between logical operators and bitwise operators?

Example:
```cpp
5 & 3
5 && 3
```

---

### 4. Explain the truth tables for:
- AND (`&`)
- OR (`|`)
- XOR (`^`)
- NOT (`~`)

---

### 5. What does the left shift (`<<`) operator do?

Example:
```cpp
5 << 2
```

Expected:
```
5 = 00000101
<<2
20 = 00010100
```

---

### 6. What does the right shift (`>>`) operator do?

Example:
```cpp
20 >> 2
```

---

### 7. What is the difference between arithmetic shift and logical shift?

---

## Intermediate Level

### 8. Check whether a number is even or odd using bitwise operators.

```cpp
bool isEven(int n) {
    return (n & 1) == 0;
}
```

---

### 9. Swap two numbers without using a temporary variable.

Using XOR:

```cpp
a ^= b;
b ^= a;
a ^= b;
```

---

### 10. Find whether the ith bit is set.

```cpp
(n >> i) & 1
```

---

### 11. Set the ith bit.

```cpp
n |= (1 << i);
```

---

### 12. Clear the ith bit.

```cpp
n &= ~(1 << i);
```

---

### 13. Toggle the ith bit.

```cpp
n ^= (1 << i);
```

---

### 14. Count the number of set bits.

Methods:
- Loop
- Brian Kernighan's Algorithm
- Built-in functions

---

### 15. Explain Brian Kernighan's Algorithm.

```cpp
while (n) {
    n &= (n - 1);
    count++;
}
```

Time Complexity:
```
O(number of set bits)
```

---

### 16. Check whether a number is a power of 2.

```cpp
n > 0 && (n & (n - 1)) == 0
```

---

### 17. Find the only non-repeating element in an array where every other element appears twice.

Using XOR.

---

### 18. Find two non-repeating numbers in an array.

Hint:
- XOR all elements
- Find rightmost set bit
- Divide into two groups

---

### 19. Reverse bits of an integer.

---

### 20. Count leading zeros.

---

## Advanced Level

### 21. Find the missing number using XOR.

Example:
```
1 2 4 5
Missing = 3
```

---

### 22. Find the unique element where every other element appears three times.

Hint:
- Count bits
- Modulo 3

---

### 23. Explain XOR properties.

- a ^ a = 0
- a ^ 0 = a
- XOR is associative
- XOR is commutative

---

### 24. Why does XOR swapping work?

Explain mathematically.

---

### 25. Generate all subsets using bit masking.

Example:
```
{1,2,3}

000
001
010
011
100
101
110
111
```

---

### 26. Solve the Traveling Salesman Problem using Bitmask DP.

---

### 27. Explain Bitmask Dynamic Programming.

Applications:
- TSP
- Assignment Problem
- State Compression

---

### 28. Explain subset masks.

Questions:
- Iterate over all subsets
- Iterate over submasks

Example:

```cpp
for (int mask = 0; mask < (1 << n); mask++)
```

---

### 29. Iterate over all submasks.

```cpp
for (int sub = mask; sub; sub = (sub - 1) & mask)
```

---

### 30. What is the lowest set bit?

Formula:

```cpp
n & (-n)
```

---

### 31. Remove the lowest set bit.

```cpp
n & (n - 1)
```

---

### 32. Find the position of the rightmost set bit.

Methods:
- Loop
- Built-in functions

---

### 33. Explain Gray Code.

Properties:
- Adjacent numbers differ by one bit.

Formula:

```cpp
gray = n ^ (n >> 1);
```

---

### 34. Explain XOR Basis.

Applications:
- Maximum XOR
- Linear Basis

---

### 35. Maximum XOR of two numbers in an array.

Methods:
- Trie
- XOR Basis

---

### 36. Maximum XOR Subarray.

---

### 37. Explain bit manipulation tricks used in competitive programming.

Examples:
- Fast multiplication/division by powers of two
- Power of 2 checks
- Bit masking

---

### 38. Explain bit fields in C.

---

### 39. Explain signed vs unsigned shifts.

---

### 40. Why is `~n == -(n+1)` in two's complement?

Explain with binary representation.

---

## Coding Interview Problems

### Easy
- Number of 1 Bits
- Reverse Bits
- Power of Two
- Missing Number
- Single Number
- Counting Bits

---

### Medium
- Single Number II
- Single Number III
- Maximum XOR of Two Numbers
- Subsets
- Gray Code
- Bitwise AND of Numbers Range

---

### Hard
- Maximum XOR Subarray
- Minimum XOR Pair
- TSP using Bitmask DP
- SOS DP
- XOR Basis Problems

---

# Common Interview Tricks

- `n & 1` → Check odd/even
- `n & (n - 1)` → Remove lowest set bit
- `n & (-n)` → Lowest set bit
- `n | (1 << i)` → Set bit
- `n & ~(1 << i)` → Clear bit
- `n ^ (1 << i)` → Toggle bit
- `(n >> i) & 1` → Check ith bit
- `n << k` → Multiply by `2^k`
- `n >> k` → Divide by `2^k`
- `n ^ n = 0`
- `n ^ 0 = n`

---

# Frequently Asked FAANG Questions

1. Single Number
2. Single Number II
3. Single Number III
4. Missing Number
5. Counting Bits
6. Reverse Bits
7. Maximum XOR of Two Numbers
8. Maximum XOR Subarray
9. Subsets using Bitmask
10. Traveling Salesman using Bitmask DP
11. Gray Code
12. Power of Two
13. Power of Four
14. Number of 1 Bits
15. Bitwise AND of Numbers Range
16. Sum of Two Integers (without `+`)
17. UTF-8 Validation
18. Decode XORed Array
19. XOR Queries of a Subarray
20. Find Original Array From Prefix XOR

---

# Complexity Cheat Sheet

| Operation | Time |
|-----------|------|
| Check bit | O(1) |
| Set bit | O(1) |
| Clear bit | O(1) |
| Toggle bit | O(1) |
| XOR swap | O(1) |
| Count bits (loop) | O(log n) |
| Brian Kernighan | O(number of set bits) |
| Iterate all subsets | O(2ⁿ) |
| Iterate submasks | O(3ⁿ) overall |
| Bitmask DP | O(2ⁿ × n) or problem-specific |

---

# Revision Checklist

- [ ] AND, OR, XOR, NOT
- [ ] Left Shift
- [ ] Right Shift
- [ ] Even/Odd
- [ ] Set/Clear/Toggle Bit
- [ ] Check ith Bit
- [ ] Count Set Bits
- [ ] Brian Kernighan Algorithm
- [ ] Power of Two
- [ ] Missing Number
- [ ] Single Number
- [ ] Reverse Bits
- [ ] Gray Code
- [ ] Bitmasking
- [ ] Subset Generation
- [ ] Submask Enumeration
- [ ] XOR Tricks
- [ ] Maximum XOR
- [ ] Bitmask DP
- [ ] Traveling Salesman Problem
````
