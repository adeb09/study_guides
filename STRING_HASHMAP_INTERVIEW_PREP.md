# String Processing + Hash Map Interview Prep
**Target:** Senior ML Engineer | O(n) solutions | Practical/real-world problems

---

## 1. Core Patterns to Master

---

### 1.1 Frequency Counting

**When to use:** Anytime you need to know "how many times does X appear" — anagram detection, character distribution, token counts, histogram bucketing.

**Complexity:** O(n) time, O(k) space where k = alphabet/vocabulary size (often bounded, so O(1) space in practice for ASCII).

**Avoids nested loops by:** Replacing the inner "scan for match" loop with a single O(1) hash lookup.

```python
from collections import Counter

freq = Counter(s)           # one pass
freq = {}
for ch in s:
    freq[ch] = freq.get(ch, 0) + 1   # equivalent, explicit
```

**Pitfalls:**
- Forgetting that `Counter` subtraction clips at 0 — use `.subtract()` if you need signed counts.
- Comparing two `Counter` objects with `==` works correctly (including zero-count keys created by subtraction — clean them with `+counter`).
- Watch out for case sensitivity: `"A"` vs `"a"` are different keys unless you normalize first.

---

### 1.2 Sliding Window + Hash Map

**When to use:** "Longest/shortest substring with property X" — distinct chars, at-most-k repeats, exactly-k distinct tokens, anagram windows.

**Complexity:** O(n) amortized; each element enters and exits the window once.

**Avoids nested loops by:** Shrinking the window from the left in O(1) using the stored frequency/index instead of rescanning.

```python
def longest_k_distinct(s: str, k: int) -> int:
    freq, left, best = {}, 0, 0
    for right, ch in enumerate(s):
        freq[ch] = freq.get(ch, 0) + 1
        while len(freq) > k:
            left_ch = s[left]
            freq[left_ch] -= 1
            if freq[left_ch] == 0:
                del freq[left_ch]
            left += 1
        best = max(best, right - left + 1)
    return best
```

**Pitfalls:**
- Not deleting keys when count hits 0 — `len(freq)` then over-counts distinct elements.
- Using a fixed-size window when the constraint is on content, not length.
- Off-by-one when computing window size: `right - left + 1`.

---

### 1.3 Prefix/Suffix Maps

**When to use:** Subarray/substring sum problems, running totals, "find a sub-sequence that satisfies a cumulative constraint."

**Complexity:** O(n) time, O(n) space for the prefix map.

**Avoids nested loops by:** Turning "does a valid prefix ending at j exist?" into a single dict lookup at each position.

```python
def count_subarrays_with_sum(arr: list[int], target: int) -> int:
    prefix_count = {0: 1}
    running, count = 0, 0
    for x in arr:
        running += x
        count += prefix_count.get(running - target, 0)
        prefix_count[running] = prefix_count.get(running, 0) + 1
    return count
```

**For strings:** Prefix hashes (rolling hash) enable O(1) substring equality checks — useful in plagiarism detection, longest repeated substring.

**Pitfalls:**
- Forgetting to seed the map with `{0: 1}` for the "empty prefix" case.
- Using mutable state across multiple calls without resetting.

---

### 1.4 Grouping / Bucketing

**When to use:** "Group these strings by some canonical form" — anagrams by sorted key, log events by session ID, records by normalized identifier.

**Complexity:** O(n · k log k) where k = string length (due to sort); O(n) if you use a frequency-tuple key instead.

**Avoids nested loops by:** Mapping each item to its canonical key in one pass, then all grouping is implicit in the dict structure.

```python
from collections import defaultdict

def group_anagrams(words: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for w in words:
        key = tuple(sorted(w))          # O(k log k)
        # key = tuple(Counter(w).items())  # unstable ordering — don't use
        key = tuple(Counter(w)[c] for c in 'abcdefghijklmnopqrstuvwxyz')  # O(k), stable
        groups[key].append(w)
    return list(groups.values())
```

**Pitfalls:**
- Using a mutable object (list) as a dict key — use `tuple` or `frozenset`.
- Sorted key is O(k log k); for large vocabularies, the 26-char frequency tuple is strictly O(k).

---

### 1.5 Index Mapping

**When to use:** "First/last occurrence," "two-sum," "find pair with property," any problem where you need to recall *where* you saw something.

**Complexity:** O(n) time, O(n) space.

**Avoids nested loops by:** Storing the index of previously seen elements; complement lookup is O(1).

```python
def two_sum(nums: list[int], target: int) -> tuple[int, int]:
    seen = {}
    for i, n in enumerate(nums):
        complement = target - n
        if complement in seen:
            return seen[complement], i
        seen[n] = i
```

**String variant:** First non-repeating character — first pass builds freq map, second pass finds first with count 1. Still O(n), no nested loop.

**Pitfalls:**
- Storing the value instead of the index (common mistake).
- Not handling the case where the same element maps to both slots (e.g., `[3, 3]` with target 6).

---

## 2. Realistic Interview-Style Problems

---

### Problem 1 — Log Parsing: First Error Per Service

**Prompt:**
Given a list of log lines in the format `"TIMESTAMP SERVICE LEVEL MESSAGE"`, return a dict mapping each service name to the timestamp of its **first ERROR-level log entry**. Lines that don't contain `ERROR` should be ignored.

```
Input:
[
  "1712000001 auth INFO user_login",
  "1712000002 payment ERROR timeout",
  "1712000003 auth ERROR invalid_token",
  "1712000004 payment ERROR retry_failed",
]

Output:
{"payment": "1712000002", "auth": "1712000003"}
```

**Thought process:**
1. Recognize: single scan, build a dict keyed by service, value = first ERROR timestamp.
2. Only insert if key not already present → naturally captures "first."
3. No sorting needed because logs arrive in time order.

```python
def first_error_per_service(logs: list[str]) -> dict[str, str]:
    first_error: dict[str, str] = {}
    for line in logs:
        parts = line.split(maxsplit=3)
        timestamp, service, level, *_ = parts
        if level == "ERROR" and service not in first_error:
            first_error[service] = timestamp
    return first_error
```

**Why hashmap:** "First occurrence per group" is a canonical dict-insert-if-absent pattern. Without it you'd need a nested scan over all prior lines for each new error.

**Complexity:** O(n) time, O(s) space where s = number of unique services.

**Edge cases:**
- Lines with fewer than 4 tokens (malformed) — add `if len(parts) < 4: continue`.
- Same service, same timestamp — irrelevant since we skip if key exists.

---

### Problem 2 — Deduplication: Normalize and Deduplicate User Records

**Prompt:**
You have a list of user identifier strings that may contain inconsistent casing, extra whitespace, or dashes/underscores. Normalize each identifier (lowercase, strip, replace `-` with `_`) and return a list of **unique normalized identifiers** in **first-seen order**.

```
Input:
["User_123", " user-123 ", "USER_123", "Admin-01", "admin_01", "new_user"]

Output:
["user_123", "admin_01", "new_user"]
```

**Thought process:**
1. Normalize = lowercase + strip + replace `-` with `_`.
2. Use an ordered `dict` (Python 3.7+ dicts are insertion-ordered) as a seen-set.
3. Single pass: normalize → check → insert.

```python
def deduplicate_users(ids: list[str]) -> list[str]:
    seen: dict[str, None] = {}
    for raw in ids:
        normalized = raw.strip().lower().replace("-", "_")
        if normalized not in seen:
            seen[normalized] = None
    return list(seen.keys())
```

**Why hashmap:** A `set` gives O(1) membership but loses insertion order. A dict preserves order while providing O(1) lookup. An O(n²) approach would scan all prior outputs for each new element.

**Complexity:** O(n · k) time where k = avg identifier length (for string ops), O(n) space.

**Edge cases:**
- Empty string after normalization — decide whether to include or skip.
- Unicode identifiers — `str.casefold()` is more correct than `.lower()` for non-ASCII.

---

### Problem 3 — Matching/Reconciliation: Token Invoice Reconciliation

**Prompt:**
You are given two lists of transaction IDs: `submitted` (what a client claims to have sent) and `processed` (what your system recorded). Return a dict with three keys:
- `"matched"`: IDs present in both
- `"missing"`: IDs in `submitted` but not in `processed`
- `"extra"`: IDs in `processed` but not in `submitted`

```
Input:
submitted = ["txn_001", "txn_002", "txn_003", "txn_004"]
processed = ["txn_002", "txn_003", "txn_005"]

Output:
{
  "matched": ["txn_002", "txn_003"],
  "missing": ["txn_001", "txn_004"],
  "extra":   ["txn_005"]
}
```

**Thought process:**
1. Build a `set` (or `frozenset`) from `processed` for O(1) membership.
2. Single pass over `submitted` → partition into matched/missing.
3. Extra = processed_set - submitted_set.

```python
def reconcile(submitted: list[str], processed: list[str]) -> dict[str, list[str]]:
    processed_set = set(processed)
    submitted_set = set(submitted)
    return {
        "matched": [t for t in submitted if t in processed_set],
        "missing": [t for t in submitted if t not in processed_set],
        "extra":   [t for t in processed if t not in submitted_set],
    }
```

**Why hashmap:** Set membership is O(1). Naive nested-loop reconciliation is O(n·m). At scale (millions of transactions), this is the difference between seconds and hours.

**Complexity:** O(n + m) time, O(n + m) space.

**Edge cases:**
- Duplicate IDs within a list (client sends same txn twice) — clarify whether to deduplicate first.
- Case sensitivity of IDs — normalize before building sets.

---

### Problem 4 — Sliding Window: Longest Pipeline Segment Without Repeated Stage

**Prompt:**
A data pipeline is described as a string of stage codes (single characters). Find the length of the **longest contiguous segment** with no repeated stage code.

```
Input: "abcadefg"
Output: 7   # "bcadefg" has 7 unique stages

Input: "aabcde"
Output: 5   # "abcde"
```

**Thought process:**
1. Classic sliding window + last-seen-index map.
2. On seeing a repeated char, jump `left` to `max(left, last_seen[ch] + 1)` — don't just increment by 1.
3. `max` is critical: avoids moving `left` backwards when a character appeared before the current window.

```python
def longest_no_repeat(s: str) -> int:
    last_seen: dict[str, int] = {}
    left = best = 0
    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1
        last_seen[ch] = right
        best = max(best, right - left + 1)
    return best
```

**Why hashmap:** Stores the exact last position of each character. Without it, finding "where was this char last seen in the current window?" requires a backward scan.

**Complexity:** O(n) time, O(min(n, alphabet_size)) space.

**Pitfalls:**
- Using `left = last_seen[ch] + 1` without the `>= left` guard — this can shrink the window backwards.

---

### Problem 5 — Anagram Detection in Stream: Sliding Anagram Window

**Prompt:**
Given a long text string `text` and a pattern string `pattern`, return all **starting indices** in `text` where an anagram of `pattern` begins.

```
Input: text="cbaebabacd", pattern="abc"
Output: [0, 6]   # "cba" at 0, "bac" at 6
```

**Thought process:**
1. Build a frequency map for `pattern`.
2. Maintain a sliding window of size `len(pattern)` over `text`.
3. Track how many character counts are currently "satisfied" (`matches` counter).
4. Slide right: add new char, check if it reaches required count → increment `matches`. Slide left: remove old char, check if it drops below → decrement `matches`.

```python
from collections import Counter

def find_anagrams(text: str, pattern: str) -> list[int]:
    if len(pattern) > len(text):
        return []

    need = Counter(pattern)
    window = Counter(text[:len(pattern)])
    matches = sum(1 for c in need if window[c] == need[c])
    k = len(pattern)
    result = [0] if matches == len(need) else []

    for i in range(k, len(text)):
        # Add right character
        rc = text[i]
        if rc in need:
            if window[rc] == need[rc]:
                matches -= 1
            window[rc] += 1
            if window[rc] == need[rc]:
                matches += 1

        # Remove left character
        lc = text[i - k]
        if lc in need:
            if window[lc] == need[lc]:
                matches -= 1
            window[lc] -= 1
            if window[lc] == need[lc]:
                matches += 1

        if matches == len(need):
            result.append(i - k + 1)

    return result
```

**Why hashmap:** Avoids re-sorting the window on every slide (which would be O(k log k) per step → O(nk log k) total). The `matches` counter reduces window comparison to O(1) per step.

**Complexity:** O(n + k) time, O(k) space.

---

### Problem 6 — Event Stream: Top-K Most Frequent Events in a Window

**Prompt:**
You receive a stream of event strings. After processing all events, return the **top-k most frequent event types**. If there's a tie, return them in lexicographic order.

```
Input: events=["login","click","login","scroll","click","login"], k=2
Output: ["login", "click"]   # login=3, click=2, scroll=1
```

**Thought process:**
1. Frequency count in one pass.
2. Sort by `(-count, name)` to handle tie-breaking.
3. Slice top-k.

```python
from collections import Counter

def top_k_events(events: list[str], k: int) -> list[str]:
    freq = Counter(events)
    return [name for name, _ in sorted(freq.items(), key=lambda x: (-x[1], x[0]))[:k]]
```

**Why hashmap:** A single-pass Counter replaces the nested loop of "for each unique event, count occurrences."

**Complexity:** O(n + u log u) where u = unique events; O(n) for counting, O(u log u) for sort.

**Pitfalls:**
- `Counter.most_common(k)` doesn't guarantee stable tie-breaking order — use explicit sort if lexicographic order matters.

---

### Problem 7 — Matching: Find All Duplicate Record Keys

**Prompt:**
Given a list of records as strings `"key:value"`, find all keys that appear **more than once**. Return them sorted.

```
Input: ["user:alice", "order:123", "user:bob", "order:456", "user:alice", "session:xyz", "order:123"]
Output: ["order", "user"]
```

**Thought process:**
1. Parse key from each record.
2. Count key frequencies.
3. Filter and sort keys with count > 1.

```python
from collections import Counter

def find_duplicate_keys(records: list[str]) -> list[str]:
    keys = (r.split(":", 1)[0] for r in records)
    freq = Counter(keys)
    return sorted(k for k, v in freq.items() if v > 1)
```

**Why hashmap:** The naive approach compares each record's key against all others — O(n²). Counter builds the frequency table in O(n).

**Complexity:** O(n) time, O(u) space where u = unique keys.

**Edge cases:**
- Records without `:` — `split(":", 1)` returns a single-element list; handle with `parts = r.split(":", 1); if len(parts) < 2: continue`.
- Keys with embedded colons — `split(":", 1)` correctly handles this by splitting only on the first colon.

---

## 3. Follow-Up Variations

---

### Follow-Up 1 — Problem 1 (Log Parsing): Streaming / Online Processing

**Original:** Process a list of logs in memory.

**Follow-up:** "Logs arrive as a live stream. You can't store all logs. Return updated first-error state after each batch."

**Evolution:**
- Maintain the `first_error` dict as persistent state across batches.
- Process each batch incrementally; don't re-process old data.

```python
class FirstErrorTracker:
    def __init__(self):
        self.first_error: dict[str, str] = {}

    def ingest(self, logs: list[str]) -> None:
        for line in logs:
            parts = line.split(maxsplit=3)
            if len(parts) < 3:
                continue
            timestamp, service, level, *_ = parts
            if level == "ERROR" and service not in self.first_error:
                self.first_error[service] = timestamp

    def get_state(self) -> dict[str, str]:
        return dict(self.first_error)
```

**Memory:** O(s) — only stores one entry per service. Bounded even if the stream is infinite.

**Scale extension:** For distributed ingestion (multiple log shards), each shard runs its own tracker, then you merge by taking the **minimum timestamp per service** across shards. This is a commutative, associative merge — suitable for MapReduce or Spark `reduceByKey`.

---

### Follow-Up 2 — Problem 3 (Reconciliation): Fuzzy Matching

**Original:** Exact string match reconciliation.

**Follow-up:** "Some IDs have a single-character typo. Count IDs in `submitted` that have a near-match (edit distance ≤ 1) in `processed`."

**Evolution:**
- Naive: O(n·m·k) — too slow for large lists.
- Efficient: For edit distance ≤ 1, generate all possible 1-edit variants of each submitted ID and check set membership.

```python
def one_edit_variants(s: str) -> set[str]:
    """All strings reachable from s by one insertion, deletion, or substitution."""
    variants = {s}
    alpha = "abcdefghijklmnopqrstuvwxyz0123456789_"
    for i in range(len(s)):
        variants.add(s[:i] + s[i+1:])                           # deletion
        for c in alpha:
            variants.add(s[:i] + c + s[i+1:])                   # substitution
            variants.add(s[:i] + c + s[i:])                     # insertion
    variants.add(s + c for c in alpha)                           # append
    return variants

def fuzzy_reconcile(submitted: list[str], processed: list[str]) -> dict[str, list[str]]:
    processed_set = set(processed)
    matched, missing = [], []
    for txn in submitted:
        if txn in processed_set or (one_edit_variants(txn) & processed_set):
            matched.append(txn)
        else:
            missing.append(txn)
    return {"matched": matched, "missing": missing}
```

**Tradeoff:** `one_edit_variants` generates O(k · |alpha|) variants per ID. For k=20 and |alpha|=37, that's ~1500 variants — still O(n·k·|alpha|) but with small constants. For very large alphabets or longer edit distances, use BK-trees or Levenshtein automata.

---

### Follow-Up 3 — Problem 4 (Sliding Window): Memory-Constrained / Fixed-Alphabet

**Original:** Find the longest substring with no repeated character.

**Follow-up:** "The input can be 10GB — you can't load it into memory. Process it as a character stream. Also: the alphabet is only {A-Z}, so optimize for that."

**Evolution:**
- Stream processing: maintain the same `last_seen` dict and `left` pointer; process one char at a time.
- Fixed alphabet: swap dict for a `bytearray` of size 26 — same O(1) access, lower memory overhead and better cache locality.

```python
def longest_no_repeat_stream(stream) -> int:
    last_seen = bytearray(b'\xff' * 26)  # 0xFF = "not seen"
    left = best = 0
    for right, ch in enumerate(stream):
        idx = ord(ch) - ord('A')
        if last_seen[idx] != 0xFF and last_seen[idx] >= left:
            left = last_seen[idx] + 1
        last_seen[idx] = right & 0xFF   # for very long streams, use int array
        best = max(best, right - left + 1)
    return best
```

**For truly unbounded streams** where `right` overflows a byte: use a plain `int` array of size 26, or maintain a deque of active positions instead of absolute indices.

---

## 4. LeetCode Problems to Revisit

| # | Problem | Pattern | Difficulty | Why It's Relevant | Key Insight |
|---|---------|---------|------------|-------------------|-------------|
| 1 | **Two Sum** | Index mapping | Easy | Foundational hash-lookup-instead-of-nested-scan | Store `val → index`; look up `target - val` |
| 49 | **Group Anagrams** | Grouping / canonical key | Medium | Exact match for "bucket strings by canonical form" | Frequency-tuple key is O(k), sort key is O(k log k) |
| 3 | **Longest Substring Without Repeating Characters** | Sliding window + last-seen index | Medium | Canonical window problem; tests left-pointer jump logic | Jump `left` to `max(left, last_seen[ch] + 1)` |
| 438 | **Find All Anagrams in a String** | Fixed-size sliding window + freq map | Medium | Directly mirrors Problem 5 above | Track `matches` count to avoid full map comparison each step |
| 76 | **Minimum Window Substring** | Variable sliding window + freq map | Hard | Hardest variant of the window pattern | Same `matches` counter technique; shrink from left when valid |
| 128 | **Longest Consecutive Sequence** | Set membership | Medium | Shows set-as-hash-map for O(1) "is neighbor present?" | Only start counting from sequence heads (`n-1 not in set`) |
| 560 | **Subarray Sum Equals K** | Prefix sum + count map | Medium | Core prefix-sum-in-hash-map pattern; transfers directly to strings | `prefix_count[running - k]` gives valid subarrays ending here |
| 387 | **First Unique Character in a String** | Two-pass frequency count | Easy | Tests "build freq map, then query it" discipline | Two O(n) passes beats one O(n²) pass |
| 242 | **Valid Anagram** | Frequency comparison | Easy | Warm-up for all frequency-map problems | Counter subtraction or sorted-key trick |
| 340 | **Longest Substring with At Most K Distinct Characters** | Sliding window + freq map | Medium | Generalizes Problem 4; tests window with a numeric constraint | Delete key when count hits 0 to keep `len(freq)` accurate |
| 567 | **Permutation in String** | Fixed-size window + freq map | Medium | Simpler variant of 438; good warmup | Same `matches` counter; simpler since window size is fixed |
| 525 | **Contiguous Array** | Prefix sum map (encoded string) | Medium | Prefix-sum technique applied to a string of 0s and 1s | Map 0 → -1, find longest subarray with prefix sum = 0 |

---

## 5. Common Interview Traps

---

### Trap 1: Accidentally O(n²) — Not Recognizing the Inner Scan

**Bad:**
```python
for i in range(len(s)):
    for j in range(i, len(s)):       # hidden O(n²)
        if s[i] == s[j]:
            ...
```

**Fix:** Ask: "Am I scanning backward or inward for a match?" If yes, that match should be a dict lookup.

---

### Trap 2: Shrinking Sliding Window Incorrectly

**Bad:**
```python
if ch in last_seen:
    left = last_seen[ch] + 1   # WRONG: can move left backward
```

**Fix:**
```python
if ch in last_seen and last_seen[ch] >= left:
    left = last_seen[ch] + 1
```

The `>= left` guard prevents the window from contracting when the duplicate is outside the current window.

---

### Trap 3: Forgetting to Delete Zero-Count Keys

**Bad:**
```python
freq[ch] -= 1
# later...
if len(freq) > k:   # WRONG: zero-count keys still count
```

**Fix:**
```python
freq[ch] -= 1
if freq[ch] == 0:
    del freq[ch]
```

Or use `Counter` and clean with `+counter` (which drops non-positive counts).

---

### Trap 4: Using Sorted Key When Frequency Tuple Suffices

**Bad pattern:** `key = tuple(sorted(word))` — O(k log k) per word.

**Better:** `key = tuple(Counter(word)[c] for c in 'abcdefghijklmnopqrstuvwxyz')` — O(k).

Only matters at scale, but signals awareness of hidden log factors.

---

### Trap 5: Misusing `Counter.most_common()` for Tie-Breaking

```python
Counter(events).most_common(k)   # tie-breaking is ARBITRARY (heap-based)
```

**Fix:** Explicit sort:
```python
sorted(freq.items(), key=lambda x: (-x[1], x[0]))[:k]
```

---

### Trap 6: Not Seeding the Prefix Map

**Bad:**
```python
prefix_count = {}          # misses the case where the prefix itself equals target
running = 0
for x in arr:
    running += x
    count += prefix_count.get(running - target, 0)
    ...
```

**Fix:**
```python
prefix_count = {0: 1}      # empty prefix has sum 0, count 1
```

---

### Trap 7: Using List for Deduplication (O(n²))

```python
seen = []
for item in items:
    if item not in seen:   # O(n) scan every time → O(n²) total
        seen.append(item)
```

**Fix:** `seen = set()` or `seen = {}` — O(1) membership.

---

### Trap 8: Not Handling Malformed Input

In parsing problems (log lines, records), always guard against:
- Fewer tokens than expected: `if len(parts) < expected: continue`
- Empty strings after normalization: `if not normalized: continue`
- Case sensitivity: normalize before inserting into dict

---

## 6. Mental Framework for Solving

---

### Step 1: Identify the Pattern (30 seconds)

Ask these questions in order:
1. **"Do I need to count something?"** → Frequency map (`Counter`)
2. **"Do I need to find something I saw before?"** → Index map or set
3. **"Do I need to group things by shared structure?"** → Canonical key + `defaultdict(list)`
4. **"Is the constraint on a contiguous window?"** → Sliding window + freq map
5. **"Is there a running total that needs to hit a target?"** → Prefix sum map

If two or more apply, they often compose (e.g., sliding window *uses* a freq map internally).

---

### Step 2: Derive the O(n) Solution

1. **Name your hash map:** What are the keys? What are the values? Say it out loud.
   - "I'll map `character → last seen index`"
   - "I'll map `prefix_sum → count of times seen`"

2. **Identify the single pass:** What do you do at each step?
   - Update the map
   - Query the map
   - Update the answer

3. **Identify what you're *avoiding*:** What nested loop does the map replace?

---

### Step 3: Code with Discipline

```
1. Handle edge cases first (empty input, k=0, single element)
2. Initialize data structures explicitly (seed prefix map, set left=0)
3. Write the main loop — no clever tricks yet
4. Add the answer tracking (max/min/count/collect)
5. Return
```

---

### Step 4: Verify with Examples

- **Happy path:** Does it work on the example?
- **Edge case:** Empty string, single character, all same characters, all unique.
- **Boundary:** What happens at index 0? At the last element?

---

### Step 5: Communicate Tradeoffs

"This is O(n) time and O(k) space where k is the alphabet/vocabulary size. If k is bounded (e.g., ASCII), space is effectively O(1). The alternative naive approach would be O(n²) because..."

For follow-ups:
- **Scaling:** "For streaming data, I'd maintain the state object across batches — the per-batch cost is still O(batch_size)."
- **Memory pressure:** "I could swap the dict for a fixed-size array indexed by character ordinal — same complexity, better cache locality."
- **Distributed:** "This pattern is embarrassingly parallelizable — each shard computes a partial map, then we merge (merge strategy depends on the aggregation: min for first-seen, sum for counts)."

---

## 7. Stretch Concepts

---

### `Counter` vs `defaultdict` vs `dict`

| | `Counter` | `defaultdict(int)` | `dict` |
|---|---|---|---|
| Missing key | Returns 0 | Returns default | Raises `KeyError` |
| Arithmetic | Full support (`+`, `-`, `&`, `\|`) | Manual | Manual |
| `most_common()` | Built-in (heap) | No | No |
| Subtraction clipping | Clips at 0 (use `.subtract()` for signed) | No | No |
| When to use | Frequency problems, anagram comparison | When you need auto-init with non-int default | When you want explicit control over missing keys |

**Rule of thumb:** Use `Counter` for frequency counting. Use `defaultdict(list)` for grouping. Use plain `dict` when you need to distinguish "key not present" from "key with zero value."

---

### Rolling Hash / String Encoding

**Use when:** You need to compare substrings in O(1) without storing them explicitly, or you need a hashable key for a window.

```python
# Rabin-Karp style rolling hash
BASE, MOD = 31, 10**9 + 7

def rolling_hash(s: str) -> int:
    h = 0
    for ch in s:
        h = (h * BASE + ord(ch)) % MOD
    return h
```

**When it matters:** Longest repeated substring, plagiarism detection, substring deduplication on large texts. For most interview problems, `tuple(Counter(window))` or direct string slicing is sufficient and clearer.

**Pitfall:** Hash collisions — always verify matches with direct string comparison when correctness is critical.

---

### Trie

**Use when:** The problem involves prefix queries across many strings — autocomplete, prefix-based routing, longest common prefix across a set.

**When NOT to use:** Most frequency/grouping/window problems don't need a Trie. If you find yourself reaching for one in a "no nested loops" problem, pause and ask if a simple dict grouping solves it instead.

```python
# Minimal Trie — only needed if prefix queries are the core requirement
from collections import defaultdict

def TrieNode():
    return defaultdict(TrieNode)

trie = TrieNode()
for word in words:
    node = trie
    for ch in word:
        node = node[ch]
    node['#'] = True  # end-of-word marker
```

---

### When NOT to Over-Engineer

| Situation | Don't reach for | Use instead |
|---|---|---|
| Grouping strings by structure | Trie | `defaultdict(list)` with canonical key |
| Checking if two strings are anagrams | Sorting + compare | `Counter(a) == Counter(b)` |
| Finding first duplicate | Bloom filter | `set` — unless truly memory-constrained |
| Top-k from frequency map | Segment tree | `Counter.most_common(k)` or `heapq.nlargest` |
| Substring search | Rolling hash | `str.find()` for small inputs; KMP/rolling hash only at scale |

**Interview signal:** Reaching for a complex data structure when a `dict` suffices suggests either over-engineering or unfamiliarity with Python's standard library. Know when simple is correct.

---

*Last updated: April 2026 | Target: Senior ML Engineer string + hashmap interview*
