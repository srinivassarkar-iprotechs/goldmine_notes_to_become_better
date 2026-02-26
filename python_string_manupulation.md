# 🧠 1. The Core Mental Model

Every string problem is one of these:

1. **Normalize** (clean input)
2. **Split** (extract structure)
3. **Transform** (change format)
4. **Validate** (check format)
5. **Reconstruct** (build output)

If you master those 5 moves, you can parse 80% of DevOps text outputs.

---

# 🔹 2. The 20% Methods You’ll Use 80% of the Time

## 🧼 Cleaning & Normalizing

```python
.strip()        # remove outer whitespace
.lstrip()
.rstrip()

.lower()
.upper()

.replace(old, new)

.split()        # auto handles multiple spaces
```

🔥 Pro move:

```python
"-".join(text.strip().lower().split())
```

Normalizes messy user input into slug format.

---

## ✂️ Splitting (Most Important Skill)

### Split once

```python
key, value = line.split("=", 1)
```

The `1` is critical — prevents breaking values containing `=`.

---

### Split from the right

```python
name, tag = image.rsplit(":", 1)
```

Use `rsplit()` when suffix is predictable.

Elite habit.

---

### Split and unpack

```python
level, date, time, *message = log.split()
```

`*message` collects the rest dynamically.

That’s senior-level Python thinking.

---

## 🔗 Joining

```python
"_".join(parts)
" ".join(words)
```

If you’re concatenating in a loop — stop. Use `join()`.

---

## 🎯 Prefix / Suffix Handling (Modern Python)

```python
.removeprefix("v")
.removesuffix(".gz")
.startswith("ERROR")
.endswith(".tar.gz")
```

These make intent obvious.

---

## 🔍 Searching

```python
if "ERROR" in log:
text.find("api")
```

Use `in` for boolean checks.
Use `.find()` only if you need index.

---

# 🔹 3. DevOps-Specific Patterns

## 🐳 Docker Image Parsing

```
registry/repo/image:tag
```

Correct pattern:

```python
name, tag = image.rsplit(":", 1)
parts = name.split("/")
```

Never hardcode indexes.

---

## 🪵 Log Parsing Pattern

```python
for line in logs.splitlines():
    parts = line.split()
```

Never use `.count("ERROR")` in real logs.
Always parse structured tokens.

---

## 🌍 URL Parsing (Professional Way)

```python
from urllib.parse import urlparse
urlparse(url).netloc
```

Do not manually slice URLs in production code.

---

## 🔐 Masking Secrets

Position-based masking:

```python
def mask(s, keep_start=4, keep_end=3):
    return s[:keep_start] + "*"*(len(s)-keep_start-keep_end) + s[-keep_end:]
```

Reusable. Clean. Safe.

---

# 🔹 4. Validation Patterns

## Semantic Version Check

```python
def is_semver(v):
    parts = v.split(".")
    return len(parts) == 3 and all(p.isdigit() for p in parts)
```

---

## IP Detection (Manual)

```python
token.count(".") == 3 and all(p.isdigit() for p in token.split("."))
```

---

# 🔹 5. When to Use Regex

Use regex only when:

* Pattern is flexible
* Structure is inconsistent
* You need extraction from messy logs

Example:

```python
import re
re.findall(r"\d+\.\d+\.\d+\.\d+", text)
```

Do NOT default to regex if simple split works.

Senior engineers prefer readable solutions.

---

# 🔹 6. Dangerous Anti-Patterns (You Were Doing Some)

❌ Hardcoded slicing:

```python
text[8:23]
```

Breaks instantly when input changes.

---

❌ Assuming fixed length:

```python
filename[-6:]
```

Fragile automation.

---

❌ Multiple `.split()` calls on same string:

```python
log.split()[0]
log.split()[1]
```

Wasteful. Store once.

---

# 🔹 7. Performance & Engineering Habits

### Cache split

Bad:

```python
log.split()[0]
log.split()[1]
```

Good:

```python
parts = log.split()
```

---

### Always `.strip()` after parsing CLI outputs

Because real-world tools output:

```
value = something\n
```

Whitespace kills automation.

---

### Parse from the right when suffix fixed

Docker tags
File extensions
Kubernetes pod suffix

Use:

```python
rsplit()
parts[-1]
```

---

# 🔹 8. Real DevOps Scenarios Where This Matters

You will use this for:

* Parsing `kubectl get pods`
* Reading `.env` files
* Transforming Git branch names in CI
* Generating dynamic Terraform variables
* Writing small CLI tools with Typer
* Masking secrets in logs
* Extracting version numbers from tags
* Validating input parameters in automation scripts

If your string skills are weak, your automation will be brittle.

---

# 🔥 9. Mastery Drill (Do This Daily)

Take any CLI output:

```
docker images
kubectl get pods
git branch
```

Pipe it to a file.

Parse it using only:

* split
* join
* strip
* unpacking
* rsplit

Do this for 30 days.

You’ll become dangerous.

---

# 🏆 Final Mental Model

When you see a string, ask:

1. What is the delimiter?
2. Is the variable part at the start or end?
3. Can the format change?
4. Should I split from left or right?
5. Should this become a reusable function?

That’s automation thinking.

---
