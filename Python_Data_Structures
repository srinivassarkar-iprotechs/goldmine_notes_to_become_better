# 🐍 Python Data Structures
### List · Tuple · Set · Dictionary — For DSA & DevOps

---

## 📋 Quick Cheat Sheet

| Feature | List | Tuple | Set | Dict |
|---|---|---|---|---|
| Syntax | `[1,2,3]` | `(1,2,3)` | `{1,2,3}` | `{k:v}` |
| Ordered | ✅ | ✅ | ❌ | ✅ (3.7+) |
| Mutable | ✅ | ❌ | ✅ | ✅ |
| Duplicates | ✅ | ✅ | ❌ | ❌ keys |
| Indexed | ✅ | ✅ | ❌ | by key |
| Hashable | ❌ | ✅ | ❌ | ❌ |

---

## 1. LIST `[]`

> **Ordered, mutable, allows duplicates. Go-to for sequences.**

### Creation
```python
nums = [1, 2, 3, 4, 5]
mixed = [1, "hello", 3.14, True]
matrix = [[1,2],[3,4],[5,6]]        # 2D list (graphs, grids)
empty = []
from_range = list(range(10))        # [0,1,2,...,9]
```

### Core Operations — O(1) unless noted
```python
nums.append(6)          # add to end          O(1)
nums.insert(2, 99)      # insert at index     O(n)
nums.pop()              # remove last         O(1)
nums.pop(0)             # remove first        O(n)  ← slow! use deque
nums.remove(99)         # remove by value     O(n)
nums[0]                 # access by index     O(1)
nums[-1]                # last element        O(1)
len(nums)               # length              O(1)
99 in nums              # search              O(n)
nums.index(3)           # find index          O(n)
nums.count(3)           # count occurrences   O(n)
nums.reverse()          # in-place reverse    O(n)
nums.sort()             # in-place sort       O(n log n)
sorted(nums)            # returns new list    O(n log n)
```

### Slicing — KEY for DSA
```python
a = [0, 1, 2, 3, 4, 5]
a[1:4]      # [1, 2, 3]       — from index 1 to 3
a[:3]       # [0, 1, 2]       — first 3
a[3:]       # [3, 4, 5]       — from index 3 onward
a[::2]      # [0, 2, 4]       — every 2nd
a[::-1]     # [5, 4, 3, 2, 1, 0]  — REVERSE (classic trick)
a[1:5:2]    # [1, 3]          — start:stop:step
```

### List Comprehensions — Write less, do more
```python
squares = [x**2 for x in range(10)]
evens   = [x for x in range(20) if x % 2 == 0]
flat    = [x for row in matrix for x in row]    # flatten 2D list
pairs   = [(x,y) for x in [1,2] for y in [3,4]] # cartesian product
```

### Useful Tricks
```python
# Unpack
a, b, *rest = [1, 2, 3, 4, 5]    # a=1, b=2, rest=[3,4,5]

# Zip two lists
keys   = ["a", "b", "c"]
values = [1,    2,    3]
zipped = list(zip(keys, values))   # [('a',1), ('b',2), ('c',3)]

# Copy (shallow)
copy = nums[:]       # or nums.copy()

# Concatenate
combined = [1,2] + [3,4]   # [1,2,3,4]

# Repeat
zeros = [0] * 5             # [0, 0, 0, 0, 0]

# Max/Min/Sum
max(nums), min(nums), sum(nums)

# Enumerate — index + value together
for i, val in enumerate(nums):
    print(i, val)
```

### DSA Patterns with List
```python
# ── STACK (LIFO) ──────────────────────────────────
stack = []
stack.append(1)     # push
stack.append(2)
stack.pop()         # pop → 2  (O(1))
stack[-1]           # peek → 1

# ── QUEUE (FIFO) — use deque, NOT list ────────────
from collections import deque
queue = deque()
queue.append(1)         # enqueue
queue.append(2)
queue.popleft()         # dequeue → 1  (O(1) vs O(n) for list.pop(0))

# ── TWO POINTER ───────────────────────────────────
def two_sum_sorted(arr, target):
    l, r = 0, len(arr) - 1
    while l < r:
        s = arr[l] + arr[r]
        if s == target: return (l, r)
        elif s < target: l += 1
        else: r -= 1

# ── SLIDING WINDOW ────────────────────────────────
def max_sum_subarray(arr, k):
    window = sum(arr[:k])
    best = window
    for i in range(k, len(arr)):
        window += arr[i] - arr[i-k]
        best = max(best, window)
    return best

# ── PREFIX SUM ────────────────────────────────────
def build_prefix(arr):
    pre = [0] * (len(arr) + 1)
    for i, v in enumerate(arr):
        pre[i+1] = pre[i] + v
    return pre
# range sum query: pre[r+1] - pre[l]
```

### DevOps Use Cases
```python
# Parse log lines
logs = open("app.log").readlines()
errors = [l for l in logs if "ERROR" in l]

# Batch process files
import os
configs = [f for f in os.listdir("/etc/nginx") if f.endswith(".conf")]

# Pipeline stages
pipeline = ["build", "test", "scan", "deploy"]
for stage in pipeline:
    print(f"Running: {stage}")

# Store command outputs
import subprocess
result = subprocess.run(["ps", "aux"], capture_output=True, text=True)
processes = result.stdout.strip().split("\n")
```

---

## 2. TUPLE `()`

> **Ordered, immutable, hashable. Use when data shouldn't change.**

### Creation
```python
point = (3, 4)
rgb   = (255, 128, 0)
single = (42,)          # ← comma required for single element
empty  = ()
packed = 1, 2, 3        # parentheses optional
```

### Why Use Tuples?
- **Faster** than lists (fixed size, less overhead)
- **Hashable** → can be used as dict keys or in sets
- **Semantically clear** → "this data is fixed"
- **Unpacking** is idiomatic Python

### Core Operations
```python
t = (10, 20, 30, 20)
t[0]            # 10       — indexing
t[-1]           # 20       — last
t[1:3]          # (20, 30) — slicing
len(t)          # 4
t.count(20)     # 2
t.index(30)     # 2
20 in t         # True
t + (40, 50)    # (10, 20, 30, 20, 40, 50)  — new tuple
t * 2           # repeat
```

### Unpacking — Most useful feature
```python
x, y = (3, 4)
a, b, c = (1, 2, 3)

# Swap without temp variable
a, b = b, a

# Ignore values
first, *_, last = (1, 2, 3, 4, 5)    # first=1, last=5

# Function returning multiple values
def min_max(arr):
    return min(arr), max(arr)          # returns tuple

lo, hi = min_max([3,1,4,1,5,9])
```

### Named Tuples — Readable structured data
```python
from collections import namedtuple

Point   = namedtuple("Point", ["x", "y"])
Server  = namedtuple("Server", ["host", "port", "protocol"])

p = Point(3, 4)
s = Server("prod-db", 5432, "tcp")

print(p.x, p.y)          # 3 4  (readable!)
print(s.host, s.port)    # prod-db 5432

# Convert to dict
s._asdict()   # {'host': 'prod-db', 'port': 5432, 'protocol': 'tcp'}
```

### DSA Patterns with Tuple
```python
# ── HEAPS — tuples as priority items ──────────────
import heapq

# Min-heap with (priority, value) tuples
heap = []
heapq.heappush(heap, (3, "task_c"))
heapq.heappush(heap, (1, "task_a"))
heapq.heappush(heap, (2, "task_b"))

priority, task = heapq.heappop(heap)   # (1, 'task_a')

# ── GRAPH EDGES ───────────────────────────────────
edges = [(0,1), (1,2), (2,3), (0,3)]   # (from, to)
weighted = [(0,1,5), (1,2,3), (2,3,1)] # (from, to, weight)

# ── COORDINATE PROBLEMS ───────────────────────────
directions = [(0,1), (0,-1), (1,0), (-1,0)]  # right,left,down,up
for dr, dc in directions:
    nr, nc = row + dr, col + dc
    # BFS/DFS grid traversal

# ── SORTING BY MULTIPLE CRITERIA ──────────────────
students = [("Alice", 85), ("Bob", 92), ("Charlie", 85)]
students.sort(key=lambda x: (-x[1], x[0]))   # sort by score desc, name asc
```

### DevOps Use Cases
```python
# Immutable config constants
DB_CONFIG = ("localhost", 5432, "mydb")
host, port, db = DB_CONFIG

# Return status + data from functions
def check_service(url):
    # ...
    return (True, 200, "OK")           # (success, code, message)

ok, code, msg = check_service("http://api/health")

# Tuples as dict keys (lists can't do this!)
cache = {}
cache[("GET", "/api/users")] = response_data

# Named tuple for parsed log entries
from collections import namedtuple
LogEntry = namedtuple("LogEntry", ["timestamp","level","service","msg"])
entry = LogEntry("2024-01-01T10:00:00", "ERROR", "auth", "Token expired")
```

---

## 3. SET `{}`

> **Unordered, mutable, no duplicates. O(1) lookup. Math set operations.**

### Creation
```python
s = {1, 2, 3, 4}
from_list = set([1, 2, 2, 3, 3])   # {1, 2, 3}  — deduplication!
empty = set()                        # NOT {} — that's a dict!
from_str = set("hello")             # {'h','e','l','o'}
```

### Core Operations — ALL O(1) average
```python
s = {1, 2, 3}
s.add(4)            # {1,2,3,4}
s.remove(2)         # {1,3,4}  — raises KeyError if missing
s.discard(99)       # no error if missing  ← safer
s.pop()             # remove & return arbitrary element
len(s)              # size
3 in s              # True   O(1) ← huge advantage over list's O(n)
3 not in s          # False
```

### Set Math — The killer feature
```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

a | b               # UNION        {1,2,3,4,5,6,7,8}
a & b               # INTERSECTION {4, 5}
a - b               # DIFFERENCE   {1, 2, 3}  (in a, not in b)
b - a               # DIFFERENCE   {6, 7, 8}
a ^ b               # SYMMETRIC DIFF {1,2,3,6,7,8} (not in both)

a.issubset(b)       # False
a.issuperset(b)     # False
a.isdisjoint(b)     # False (they share 4,5)

# Equivalents with operators
a.union(b)
a.intersection(b)
a.difference(b)
a.symmetric_difference(b)
```

### Frozenset — Immutable set (hashable)
```python
fs = frozenset([1, 2, 3])
# Can be used as dict key or inside another set
graph = {frozenset({1,2}): "edge", frozenset({2,3}): "edge"}
```

### DSA Patterns with Set
```python
# ── DEDUPLICATION ─────────────────────────────────
arr = [1, 2, 2, 3, 3, 3, 4]
unique = list(set(arr))     # [1, 2, 3, 4]

# ── FAST LOOKUP / SEEN TRACKING ───────────────────
def has_duplicate(arr):
    return len(arr) != len(set(arr))    # O(n)

def two_sum(nums, target):              # O(n)
    seen = set()
    for n in nums:
        if target - n in seen:
            return True
        seen.add(n)
    return False

# ── VALID ANAGRAM ─────────────────────────────────
def is_anagram(s, t):
    from collections import Counter
    return Counter(s) == Counter(t)
# or: set check for unique chars
def same_chars(s, t):
    return set(s) == set(t)

# ── CYCLE DETECTION (Floyd / set method) ──────────
def has_cycle_set(head):               # linked list
    visited = set()
    node = head
    while node:
        if id(node) in visited: return True
        visited.add(id(node))
        node = node.next
    return False

# ── GRAPH: VISITED TRACKING ───────────────────────
def dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited: continue
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                stack.append(neighbor)
    return visited

# ── COMMON ELEMENTS / MISSING ELEMENTS ────────────
deployed = {"v1", "v2", "v3"}
expected = {"v1", "v2", "v3", "v4"}
missing  = expected - deployed          # {"v4"}
extra    = deployed - expected          # {}
```

### DevOps Use Cases
```python
# Unique IPs from logs
log_lines = ["192.168.1.1 GET /api", "10.0.0.1 POST /login", "192.168.1.1 GET /"]
unique_ips = {line.split()[0] for line in log_lines}

# Permission checks
user_perms  = {"read", "write"}
req_perms   = {"read", "execute"}
can_do_all  = req_perms.issubset(user_perms)        # False
missing_p   = req_perms - user_perms                # {"execute"}

# Tag deduplication in CI/CD
tags = ["python", "docker", "python", "k8s", "docker"]
unique_tags = list(set(tags))

# Find servers not in both environments
prod    = {"server1", "server2", "server3"}
staging = {"server1", "server3", "server4"}
only_prod    = prod - staging       # {"server2"}
only_staging = staging - prod       # {"server4"}
in_both      = prod & staging       # {"server1", "server3"}
```

---

## 4. DICTIONARY `{key: value}`

> **Key-value store. O(1) get/set/delete. Most versatile structure.**

### Creation
```python
d = {"name": "Alice", "age": 30}
empty = {}
from_pairs = dict([("a", 1), ("b", 2)])
from_keys  = dict.fromkeys(["x","y","z"], 0)    # {'x':0,'y':0,'z':0}

# Dict comprehension
squares = {x: x**2 for x in range(6)}           # {0:0, 1:1, 2:4, ...}
word_len = {w: len(w) for w in ["cat","dog","elephant"]}
```

### Core Operations — O(1) average
```python
d = {"a": 1, "b": 2, "c": 3}

d["a"]              # 1          — access (KeyError if missing)
d.get("z")          # None       — safe access
d.get("z", 0)       # 0          — with default ← use this!
d["d"] = 4          # add/update
del d["a"]          # delete key (KeyError if missing)
d.pop("b")          # delete + return value
d.pop("z", None)    # safe pop with default

"a" in d            # True   O(1) membership check
len(d)              # size
d.keys()            # dict_keys(['b','c','d'])
d.values()          # dict_values([2, 3, 4])
d.items()           # dict_items([('b',2),('c',3),('d',4)])

# Merge dicts
d1 = {"a": 1}
d2 = {"b": 2}
merged = {**d1, **d2}           # {'a':1, 'b':2}  ← most common
d1.update(d2)                   # in-place merge
merged = d1 | d2                # Python 3.9+
```

### Iteration Patterns
```python
config = {"host": "localhost", "port": 5432, "db": "prod"}

for key in config:                          # iterate keys
    print(key)

for val in config.values():                 # iterate values
    print(val)

for key, val in config.items():             # iterate key-value pairs
    print(f"{key} = {val}")

# Dict comprehension with filter
filtered = {k: v for k, v in config.items() if k != "db"}
```

### defaultdict & Counter — Must know
```python
from collections import defaultdict, Counter

# defaultdict — never KeyError, auto-initializes
graph   = defaultdict(list)     # adjacency list
graph[1].append(2)              # no need to check if key exists
graph[1].append(3)
# graph = {1: [2, 3]}

freq = defaultdict(int)
for char in "hello":
    freq[char] += 1             # no KeyError on first access
# freq = {'h':1, 'e':1, 'l':2, 'o':1}

# Counter — frequency counting in one shot
text  = "hello world"
count = Counter(text)           # Counter({'l':3,'o':2,'h':1,...})
count.most_common(3)            # [('l',3),('o',2),('h',1)]
count["l"]                      # 3
count["z"]                      # 0  (no KeyError)

# Counter arithmetic
c1 = Counter("aab")
c2 = Counter("abc")
c1 + c2     # Counter({'a':3, 'b':2, 'c':1})
c1 - c2     # Counter({'a':1})
c1 & c2     # min: Counter({'a':1, 'b':1})
c1 | c2     # max: Counter({'a':2, 'b':1, 'c':1})
```

### OrderedDict & LRU
```python
from collections import OrderedDict

# LRU Cache — classic DSA problem
class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache: return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, val):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = val
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)
```

### DSA Patterns with Dict
```python
# ── FREQUENCY / COUNTING ──────────────────────────
def char_freq(s):
    freq = {}
    for c in s:
        freq[c] = freq.get(c, 0) + 1
    return freq

# ── MEMOIZATION (TOP-DOWN DP) ─────────────────────
def fib(n, memo={}):
    if n in memo: return memo[n]
    if n <= 1: return n
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]

# or use functools
from functools import lru_cache
@lru_cache(maxsize=None)
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)

# ── GRAPH AS ADJACENCY LIST ───────────────────────
graph = {
    0: [1, 2],
    1: [0, 3],
    2: [0, 4],
    3: [1],
    4: [2]
}

# ── GROUPING / BUCKETING ──────────────────────────
def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        key = tuple(sorted(w))     # sorted chars as key
        groups[key].append(w)
    return list(groups.values())

group_anagrams(["eat","tea","tan","ate","nat","bat"])
# [['eat','tea','ate'], ['tan','nat'], ['bat']]

# ── TWO SUM with dict ─────────────────────────────
def two_sum(nums, target):
    seen = {}       # value → index
    for i, n in enumerate(nums):
        diff = target - n
        if diff in seen:
            return [seen[diff], i]
        seen[n] = i

# ── SLIDING WINDOW with freq map ──────────────────
def longest_unique_substr(s):
    freq = {}
    l = res = 0
    for r, c in enumerate(s):
        freq[c] = freq.get(c, 0) + 1
        while freq[c] > 1:
            freq[s[l]] -= 1
            l += 1
        res = max(res, r - l + 1)
    return res
```

### DevOps Use Cases
```python
# Config management
config = {
    "database": {"host": "db.prod", "port": 5432},
    "cache":    {"host": "redis.prod", "port": 6379},
    "app":      {"debug": False, "workers": 4}
}
db_host = config["database"]["host"]

# Environment variable parsing
import os
env = {k: v for k, v in os.environ.items() if k.startswith("APP_")}

# Process inventory tracking
processes = {}
for line in subprocess.check_output(["ps","aux"]).decode().splitlines()[1:]:
    parts = line.split()
    pid, cmd = parts[1], parts[10]
    processes[pid] = {"cmd": cmd, "cpu": parts[2], "mem": parts[3]}

# API response parsing
import json
response = json.loads(api_response_text)
status = response.get("status", "unknown")
data   = response.get("data", [])

# Deployment tracking
deployments = defaultdict(list)
deployments["prod"].append({"version": "v2.3", "time": "2024-01-01"})
deployments["staging"].append({"version": "v2.4", "time": "2024-01-02"})

# Counting log levels
from collections import Counter
log_levels = [l.split()[1] for l in open("app.log") if len(l.split()) > 1]
summary = Counter(log_levels)
# Counter({'INFO': 1500, 'WARN': 45, 'ERROR': 12})
```

---

## 5. Choosing the Right Structure

```
Need ordered sequence?
├── Will it change?
│   ├── YES → LIST
│   └── NO  → TUPLE (also use as dict key)
Need fast lookup by key?
│   └── DICT
Need uniqueness / membership test?
│   └── SET
Need frequency count?
│   └── Counter (dict subclass)
Need default values?
│   └── defaultdict
Need ordered dict + LRU?
│   └── OrderedDict
```

---

## 6. Performance Summary

| Operation | List | Tuple | Set | Dict |
|---|---|---|---|---|
| Access by index | O(1) | O(1) | ❌ | O(1) by key |
| Search (`in`) | O(n) | O(n) | **O(1)** | **O(1)** |
| Insert end | O(1) | ❌ | O(1) | O(1) |
| Insert middle | O(n) | ❌ | O(1) | O(1) |
| Delete | O(n) | ❌ | O(1) | O(1) |
| Memory | Medium | **Low** | Medium | High |

**Key insight**: Use `set` or `dict` over `list` whenever you need fast membership testing (`x in something`). This alone can turn O(n²) algorithms into O(n).

---

## 7. Power Combos

```python
# Dict of lists — adjacency list, grouping
graph = defaultdict(list)

# Dict of sets — unique neighbors
seen_per_user = defaultdict(set)

# Set of tuples — visited coordinates in grid BFS
visited = set()
visited.add((row, col))

# List of dicts — records / JSON-like data
servers = [
    {"name": "web-01", "ip": "10.0.0.1", "status": "up"},
    {"name": "web-02", "ip": "10.0.0.2", "status": "down"},
]
up_servers = [s["name"] for s in servers if s["status"] == "up"]

# Nested dict — config / tree
tree = {"val": 1, "left": {"val": 2, "left": None, "right": None}, "right": None}

# Counter + heap — top K elements
from collections import Counter
import heapq

def top_k_frequent(nums, k):
    freq = Counter(nums)
    return heapq.nlargest(k, freq.keys(), key=freq.get)

top_k_frequent([1,1,1,2,2,3], 2)    # [1, 2]
```

---

## 8. Quick Reference: Collections Module

```python
from collections import (
    Counter,        # frequency counting
    defaultdict,    # dict with default values
    OrderedDict,    # ordered dict + LRU helper
    deque,          # O(1) append/pop from both ends
    namedtuple,     # tuple with named fields
    ChainMap,       # view multiple dicts as one
)

# deque — double-ended queue
dq = deque([1,2,3])
dq.appendleft(0)    # [0,1,2,3]
dq.append(4)        # [0,1,2,3,4]
dq.popleft()        # 0  O(1)
dq.pop()            # 4  O(1)
dq.rotate(1)        # right rotate: [3,1,2]
dq.rotate(-1)       # left rotate:  [1,2,3]

# ChainMap — layered config (env > file > defaults)
from collections import ChainMap
defaults = {"debug": False, "port": 8080}
file_cfg = {"port": 9090}
env_cfg  = {"debug": True}
config   = ChainMap(env_cfg, file_cfg, defaults)
config["debug"]     # True  (from env_cfg)
config["port"]      # 9090  (from file_cfg)
```

---

*Master these and you have the foundation for 90% of DSA problems and all common DevOps scripting tasks.*
