# Python Basics — Interview Revision Notes

---

## 1. Variables

**Concept:** Variables are names bound to objects in memory (not containers). `=` binds a name to an object; the object has a type, not the variable.

**Syntax:**
```python
x = 10
a, b, c = 1, 2, 3      # multiple assignment
a = b = c = 5            # chained assignment (same object)
x, y = y, x                # swap
```

**Built-ins:** `id(obj)` → memory address | `type(obj)` → type | `is` → identity check | `del x` → removes name binding

**Common Mistakes:**
- Confusing `==` (value) with `is` (identity)
- Assuming `b = a` copies a mutable object — it just creates another reference
- Relying on small-int caching (-5 to 256) as guaranteed behavior — it's a CPython detail

---

## 2. Data Types

**Concept:** Numeric (`int`,`float`,`complex`), Sequence (`str`,`list`,`tuple`), Set (`set`,`frozenset`), Mapping (`dict`), `bool`, `NoneType`.
**Mutable:** list, dict, set, bytearray. **Immutable:** int, float, str, tuple, bool, frozenset.
Only immutable (hashable) types can be dict keys / set elements.

**Syntax:**
```python
i=10; f=3.14; c=2+3j; s="hi"; l=[1,2]; t=(1,2); st={1,2}; fs=frozenset({1,2}); d={"a":1}; b=True; n=None
```

**Built-ins:** `type(obj)` (exact type) | `isinstance(obj, type_or_tuple)` (inheritance-aware, preferred)

**Common Mistakes:**
- `type()==` instead of `isinstance()` breaks with inheritance
- Forgetting `bool` is a subclass of `int` → `{1:'a', True:'b'}` collapses to one key
- Using floats for exact decimal comparison (`0.1+0.2 != 0.3`) — use `Decimal`/`round()`
- Trying to use `list`/`dict`/`set` as dict keys → `TypeError: unhashable type`
- Assuming Python `int` overflows like C/Java — it doesn't (arbitrary precision)

---

## 3. Type Casting

**Concept:** Implicit (auto-widening in mixed arithmetic: `int+float→float`) vs Explicit (`int()`, `float()`, etc.). `int()` truncates toward zero, doesn't round.

**Syntax:**
```python
int("10"); float("3.14"); str(100); bool(0); list("hi"); tuple([1,2]); set([1,1,2]); dict([("a",1)])
chr(65); ord('A'); hex(255); oct(8); bin(5)
```

**Built-ins:** `int(x,base)`, `float(x)`, `str(x)`, `bool(x)`, `list/tuple/set/dict(iterable)` — all O(n) for iterables

**Common Mistakes:**
- `int()` truncates, doesn't round (`int(3.99)==3`)
- `int("3.14")` fails — must go through `float()` first: `int(float("3.14"))`
- `bool("False")` is `True` — any non-empty string is truthy
- `set()` removes duplicates but **loses order**
- `dict([("a",1),("a",2)])` → later duplicate key wins → `{"a":2}`
- `round(2.5)==2` (banker's rounding — rounds half to even)

---

## 4. Strings

**Concept:** Immutable sequence of Unicode chars. All "modifying" methods return NEW strings. `+=` in a loop is O(n²) — use `"".join()` instead (O(n)).

**Syntax:**
```python
s1='hello'; s2="world"; s3='''multi\nline'''; s4=r"C:\path"; s5=f"val={5+3}"
s[0:5]; s[::-1]  # reversed; s[::2]
```

**Built-ins (key ones):**
| Method | Notes |
|---|---|
| `.upper()/.lower()/.title()/.capitalize()` | return new strings |
| `.find(sub)` | -1 if not found |
| `.index(sub)` | raises `ValueError` if not found |
| `.count(sub)` | non-overlapping only |
| `.strip()/.lstrip()/.rstrip()` | trims whitespace/chars |
| `.split(sep)/.rsplit()/.splitlines()` | → list |
| `.join(iterable)` | called on separator: `"-".join([...])` |
| `.replace(old,new,count)` | |
| `.isalpha()/.isdigit()/.isalnum()/.isspace()` | validation checks; empty string → False |
| `.encode()/bytes.decode()` | str ↔ bytes |

**Common Mistakes:**
- Trying `s[0]='H'` — strings are immutable, raises `TypeError`
- `+=` concatenation in loops — O(n²); use `"".join()`
- `.find()` vs `.index()` — wrong one for the situation (silent -1 vs exception)
- `.split()` (no args) collapses whitespace; `.split(" ")` doesn't
- `.count()` doesn't count overlapping substrings
- Slicing out-of-range never errors (`s[100:200]`→`""`); direct indexing does (`s[100]`→IndexError)

---

## 5. Lists

**Concept:** Mutable, ordered, dynamic array of object references. Indexing O(1). Append amortized O(1) (over-allocation); insert/delete at arbitrary position O(n).

**Syntax:**
```python
l1=[1,2,3]; l2=list(); l3=[x for x in range(5)]; l4=list(range(5)); l5=list("hi")
```

**Built-ins:**
| Method | Purpose | Complexity | Notes |
|---|---|---|---|
| `.append(x)` | add one element to end | O(1) amortized | adding a list nests it as ONE element |
| `.extend(iter)` | add elements individually | O(k) | same as `l += iter` |
| `.insert(i,x)` | insert at index | O(n) | out-of-range index just appends |
| `.remove(x)` | remove first matching VALUE | O(n) | raises `ValueError` if missing |
| `.pop(i=-1)` | remove & RETURN by index | O(1) end / O(n) mid | only removal method that returns value |
| `.clear()` | empty the list | O(n) | |
| `.index(x)` | find index of value | O(n) | raises `ValueError` if missing |
| `.count(x)` | count occurrences | O(n) | |
| `.sort(key,reverse)` | in-place sort, returns `None` | O(n log n) | Timsort |
| `.reverse()` | in-place reverse, returns `None` | O(n) | |
| `.copy()` | SHALLOW copy | O(n) | nested mutables still shared |

**Common Mistakes:**
- `l = l.sort()` → wipes list to `None` (sort returns None)
- `.append(x)` vs `.extend(x)` confusion
- `.copy()`/`l[:]` is shallow — nested lists still shared; use `copy.deepcopy()`
- `list(set(lst))` to dedupe doesn't preserve order
- Mutating a list while iterating over it directly → skipped elements
- `l = l + [x]` in a loop instead of `.append(x)` — O(n²) vs amortized O(n)

---

## 6. Tuples

**Concept:** Immutable, ordered sequence. Hashable (usable as dict key/set element) ONLY if all elements are hashable. Mutable object inside a tuple can still be mutated in place.

**Syntax:**
```python
t1=(1,2,3); t2=1,2,3        # packing, parens optional
t3=(1,)                        # single-element -- COMMA required
t4=(1)                            # NOT a tuple, just int
a,b,c=t1; a,*rest=t1                # unpacking / star unpacking
```

**Built-ins:** `.count(x)` O(n) | `.index(x)` O(n), raises `ValueError` if missing — that's it, only 2 methods (immutable → no add/remove)

**Common Mistakes:**
- `(5)` is an int, `(5,)` is a tuple — comma makes the tuple, not parentheses
- Assuming full immutability — a list INSIDE a tuple can still be mutated
- Calling `.append()`/`.sort()` on a tuple → `AttributeError`
- Assuming any tuple can be a dict key — only if ALL elements are hashable

---

## 7. Sets

**Concept:** Mutable, unordered collection of unique hashable elements. Backed by a hash table → membership test average O(1) vs list's O(n). `frozenset` = immutable/hashable version.

**Syntax:**
```python
s1={1,2,3}; s2=set()   # {} is a DICT, not a set!
s3=set([1,1,2]); s4={x for x in range(5)}; fs=frozenset({1,2})
```

**Built-ins:**
| Method | Notes |
|---|---|
| `.add(x)` | no effect if exists |
| `.update(iter)` | adds multiple elements |
| `.remove(x)` | raises `KeyError` if missing |
| `.discard(x)` | no error if missing (safer) |
| `.pop()` | removes ARBITRARY element |
| `.clear()` | empties set |
| `.union()/|` , `.intersection()/&` , `.difference()/-` , `.symmetric_difference()/^` | set algebra, O(len1+len2) |
| `.issubset()/<=`, `.issuperset()/>=`, `.isdisjoint()` | relational checks |
| `.copy()` | shallow copy |

**Common Mistakes:**
- `{}` creates a dict, not an empty set — use `set()`
- Assuming sets preserve insertion order — they don't
- Using `.remove()` on possibly-missing value without handling `KeyError` — prefer `.discard()`
- Trying to add unhashable types (list, dict, set) → `TypeError`
- Using sets for anagram checks — sets discard frequency info; use `Counter` instead

---

## 8. Dictionaries

**Concept:** Mutable key-value store backed by a hash table. Average O(1) get/set/delete. Since Python 3.7, preserves **insertion order**. Only hashable objects can be keys.

**Syntax:**
```python
d1={"a":1,"b":2}; d2=dict(); d3=dict(a=1,b=2); d4=dict([("a",1)]); d5={k:v for k,v in d1.items()}
```

**Built-ins:**
| Method | Purpose | Notes |
|---|---|---|
| `d[key]` | access | raises `KeyError` if missing |
| `.get(key,default)` | safe access | returns default/None, no error |
| `.setdefault(key,default)` | get-or-insert | great for "group by" patterns |
| `.update(other)` | merge in place | later values win on conflict |
| `.pop(key,default)` | remove & return | `KeyError` if missing & no default |
| `.popitem()` | remove LAST-inserted pair | LIFO since 3.7 |
| `.clear()` | empty dict | |
| `.keys()/.values()/.items()` | dynamic VIEW objects, not lists | reflect live changes |
| `dict.fromkeys(iter,val)` | build dict with same value for all keys | |
| `{**d1,**d2}` / `d1 \| d2` (3.9+) | merge (right-hand wins) | |

**Common Mistakes:**
- `d[key]` on missing key → `KeyError`; use `.get()` or check `in` first
- Treating `.keys()/.values()/.items()` as lists — they're views; wrap in `list()` if needed
- Mutating dict size while iterating directly over it → `RuntimeError`
- Using mutable object as key → `TypeError: unhashable type`
- `.pop(key)` without default on missing key → `KeyError`

---

## 9. Conditional Statements

**Concept:** `if/elif/else`; no switch (until `match-case`, 3.10+). Non-bool conditions use truthiness. Falsy: `0, 0.0, "", [], {}, (), set(), None, False`. `and`/`or` short-circuit and return the actual operand, not just True/False.

**Syntax:**
```python
if x>5: ...
elif x==5: ...
else: ...
result = "even" if x%2==0 else "odd"    # ternary
if 1 < x < 20: ...                          # chained comparison
match x:
    case 10: ...
    case _: ...
```

**Key operators:** `and/or/not`, `in/not in`, `is/is not` (prefer over `==` for `None`)

**Common Mistakes:**
- `x == None` instead of `x is None`
- `x or default` when `x` could legitimately be `0`/`""`/`False` — wrongly overwritten
- `and`/`or` return the operand value, not a strict boolean
- Deep if-else nesting instead of `elif`
- Mixing tabs/spaces → `IndentationError`/`TabError`

---

## 10. Loops

**Concept:** `for` iterates via the iterator protocol (`iter()`+`next()` until `StopIteration`). `while` checks condition before each pass — no built-in do-while. `range()` is lazy, NOT a list.

**Syntax:**
```python
for i in range(5): ...
for ch in "hi": ...
i=0
while i<5:
    i+=1
for i in range(10):
    if i==5: break
    if i%2==0: continue
for i in range(5):
    ...
else:
    print("no break happened")   # runs only if loop completed w/o break
```

**Built-ins:** `range(start,stop,step)` O(1) space | `enumerate(iter,start=0)` → (index,value) | `zip(*iters)` → pairs, **stops at shortest** | `reversed(seq)` | `next(iterator,default)`

**Common Mistakes:**
- Forgetting to update loop variable in `while` → infinite loop
- Assuming `range()` is a list
- Misunderstanding `for-else` — else runs only if `break` never hit
- Assuming `zip()` pads mismatched lengths — it truncates silently
- `break`/`continue` only affect the INNERMOST loop
- Mutating a list/dict while iterating directly over it

---

## 11. Functions

**Concept:** First-class objects — assignable, passable, returnable. Each call creates a new stack frame (local namespace). Scope resolution: **LEGB** (Local→Enclosing→Global→Built-in). Default arg values evaluated ONCE at def time.

**Syntax:**
```python
def greet(name): return f"Hi {name}"
def greet2(name="Guest"): ...
def add(a,b,*args): ...              # *args -> tuple
def show(**kwargs): ...               # **kwargs -> dict
def combo(a,b,*args,c=10,**kwargs): ...
def pos_only(a,b,/,c): ...              # positional-only (3.8+)
def kw_only(a,*,b): ...                  # keyword-only
```

**Built-ins/attrs:** `func.__name__`, `func.__doc__`, `callable(obj)`, `globals()/locals()`

**Common Mistakes:**
- **Mutable default argument** (`def f(x, l=[])`) — created once, shared across calls; fix with `l=None` sentinel
- No `return` → function returns `None` implicitly
- Modifying a global var without `global` keyword → `UnboundLocalError`
- Forgetting `nonlocal` for enclosing (non-global) scope modification
- Reassigning a parameter inside a function does NOT affect caller; mutating it DOES (for mutable objects)
- `*args` must come before `**kwargs` in signature

---

## 12. Lambda Functions

**Concept:** Anonymous, single-expression function. Functionally identical to `def` (same bytecode/closures/LEGB) — NO performance advantage. Body must be ONE expression (no statements, no explicit `return`).

**Syntax:**
```python
square = lambda x: x*x
add = lambda a,b: a+b
classify = lambda x: "even" if x%2==0 else "odd"
sorted(data, key=lambda s: s[1])
list(map(lambda x: x**2, nums))
list(filter(lambda x: x%2==0, nums))
from functools import reduce
reduce(lambda a,b: a*b, nums)
```

**Common Mistakes:**
- Assuming lambdas are faster than `def` — they aren't (same bytecode)
- Overusing lambdas for complex logic — hurts readability, should be `def`
- **Late-binding closure trap** in loops: `[lambda: i for i in range(3)]` → all return `2`. Fix: `lambda i=i: i` (default arg forces early binding)
- PEP 8 discourages `f = lambda x: x+1` — use `def` if you need a name
- Trying to include statements/assignments inside lambda body → `SyntaxError`

---

## 13. List, Dict, Set Comprehensions

**Concept:** Concise syntax to build a new collection from an iterable + expression (+ optional filter). Own local scope in Python 3 (loop var doesn't leak, unlike Python 2). Generator expressions `(...)` are LAZY — don't build the full collection in memory.

**Syntax:**
```python
squares = [x**2 for x in range(5)]
evens = [x for x in range(10) if x%2==0]                    # FILTER (at end)
labeled = [("even" if x%2==0 else "odd") for x in range(5)]    # TRANSFORM (before for) -- no filtering
squares_dict = {x:x**2 for x in range(5)}
unique_lens = {len(w) for w in words}
flat = [n for row in matrix for n in row]                        # nested, left-to-right
gen = (x**2 for x in range(5))                                       # lazy generator expression
```

**Common Mistakes:**
- Confusing filtering `if` (end) with conditional `if-else` (before `for`) — filter excludes, ternary transforms all
- Over-nesting (3+ levels) — hurts readability
- Wrong order of nested `for` clauses → `NameError` (must match nested-loop order, left to right)
- Believing comprehension vars leak into enclosing scope in Python 3 — they don't (Python 2 difference)
- Using a list comprehension instead of a generator expression when the result is consumed only once — wastes memory for large sequences

---

## Quick Cross-Topic Reminders
- **Mutable:** list, dict, set, bytearray → default-arg trap, shallow-copy trap, unhashable-key trap
- **Immutable/hashable:** int, float, str, tuple(if contents hashable), frozenset, bool → safe as dict keys/set elements
- **`is` vs `==`**: identity vs value — always use `is` for `None`
- **O(1) avg lookup**: dict & set (hash table backed) vs O(n) for list membership
- **Returns `None`**: `.sort()`, `.reverse()`, `.append()`, `.extend()`, `.update()`, any function without explicit `return`
- **LEGB** scope order; `global` for module-level, `nonlocal` for enclosing-function level
