---
hide:
    - toc
    - navigation
---

# Probabilistic Data Structures for High Cardinality Scale

> The mathematics, memory mechanics, and cache locality of Bloom filters, Count-Min sketches, and HyperLogLog in production distributed systems.

## The Cost of Exactness at Scale

At low scale, deterministic accuracy is computationally cheap. Storing a million 64-bit user identifiers in a hash table requires roughly 16 megabytes of memory. Querying membership is an $O(1)$ memory dereference.

At hyperscale, exactness becomes an operational bottleneck. When a streaming system processes ten billion events per day, maintaining exact state structures collapses under memory limits and cache line thrashing:

1. Tracking unique IP addresses across a global network in a standard hash set consumes upwards of 160 gigabytes of RAM.
2. Maintaining exact top-k frequency counters for billions of search queries causes massive write contention on lock-guarded hash tables.
3. Checking whether a user ID exists in a petabyte-scale distributed database before issuing an expensive disk read incurs unacceptable I/O latency.

Probabilistic data structures resolve these bottlenecks by trading deterministic perfection for massive constant factor reductions in memory and compute. By accepting a controlled, mathematically bounded error rate, you reduce memory consumption by multiple orders of magnitude while preserving $O(1)$ performance.

## 1. Bloom Filters: Rejection at Zero I/O Cost

A Bloom filter is a space-efficient probabilistic data structure designed to answer set membership queries. When queried, it produces one of two answers:

1. **Definite Absence (False):** The element is definitively not present in the set, guaranteeing zero false negatives.
2. **Probable Presence (True):** The element is likely present in the set, subject to a mathematically bounded false positive probability.

This asymmetric guarantee is exceptionally valuable for storage engines. A false positive merely triggers an unnecessary disk read. A false negative would produce silent data corruption by reporting that an existing record does not exist.

### Mathematical Foundations

A Bloom filter represents a set of $n$ elements using a bit array of size $m$, indexed by $k$ independent, uniform hash functions.

To achieve a target false positive probability $p$ for an expected capacity $n$, the optimal bit array size $m$ is given by:

$$m = -\frac{n \ln(p)}{(\ln 2)^2}$$

The optimal number of hash functions $k$ that minimizes false positives is:

$$k = \frac{m}{n} \ln 2$$

For example, to store 10,000,000 elements with a 1% false positive probability ($p = 0.01$):

$$m \approx -\frac{10^7 \cdot (-4.605)}{0.4804} \approx 95,850,583 \text{ bits} \approx 11.42 \text{ MB}$$
$$k \approx \frac{9.585}{1} \cdot 0.693 \approx 6.64 \implies 7 \text{ hash functions}$$

Storing ten million keys in a standard hash set requires hundreds of megabytes. A Bloom filter achieves 99% rejection accuracy in less than 12 megabytes.

### Kirsch-Mitzenmacher Hash Optimization

Computing $k$ independent cryptographic hashes (such as running SHA-256 seven times) introduces significant CPU overhead. 

In production, systems utilize the Kirsch-Mitzenmacher optimization (*Less Hashing, Same Performance: Building a Better Bloom Filter*). Two independent 64-bit hash values, $h_1(x)$ and $h_2(x)$ (produced via MurmurHash3 or xxHash), can synthesize an arbitrary number of hash indices using the linear recurrence:

$$g_i(x) = (h_1(x) + i \cdot h_2(x)) \pmod m \quad \text{for } i \in [0, k-1]$$

This reduces hash evaluation cost from $k$ independent hash passes to a single 128-bit hash invocation followed by integer arithmetic.

### Production Implementation

```python
import math
import mmh3
from typing import List

class ProductionBloomFilter:
    def __init__(self, expected_elements: int, false_positive_rate: float = 0.01):
        self.n = expected_elements
        self.p = false_positive_rate

        # Calculate optimal size m and hash count k
        self.m = int(- (self.n * math.log(self.p)) / (math.log(2) ** 2))
        self.k = int(round((self.m / self.n) * math.log(2)))

        # Allocate bit array using 64-bit integers for efficiency
        self.num_words = (self.m + 63) // 64
        self.bit_array = [0] * self.num_words

    def add(self, key: str):
        # 128-bit hash yields two 64-bit hash values
        h1, h2 = mmh3.hash64(key, seed=0)

        for i in range(self.k):
            index = (h1 + i * h2) % self.m
            word_idx = index // 64
            bit_idx = index % 64
            self.bit_array[word_idx] |= (1 << bit_idx)

    def contains(self, key: str) -> bool:
        h1, h2 = mmh3.hash64(key, seed=0)

        for i in range(self.k):
            index = (h1 + i * h2) % self.m
            word_idx = index // 64
            bit_idx = index % 64
            if not (self.bit_array[word_idx] & (1 << bit_idx)):
                return False
        return True
```

### Modern Architecture: Blocked Bloom Filters for Cache Locality

Standard Bloom filters suffer from a CPU cache locality defect. Setting or testing $k$ bits across an 80MB array triggers up to $k$ independent L3 CPU cache misses per query.

Modern high throughput storage engines (such as RocksDB and Apache Parquet) implement **Blocked Bloom Filters**. The filter is partitioned into cache-line sized blocks (typically 64 bytes / 512 bits). A primary hash function selects the 64-byte block, and the $k$ bit offsets are evaluated strictly within that single cache line. This reduces memory stalls from $k$ DRAM accesses to a single cache line fetch.

## 2. Count-Min Sketch: Frequency Estimation in Fixed Space

When processing infinite event streams (such as network packet traffic, API access rates, or clickstream events), tracking exact occurrence frequencies for every key is impossible because the cardinality is unbounded.

A Count-Min Sketch estimates frequency counts using a fixed two dimensional matrix of counters defined by width $w$ (which bounds the magnitude of overestimation error) and depth $d$ (which bounds the failure probability).

```
          Index 0    Index 1    Index 2   ...   Index w-1
Row 0   [   42   ] [    0   ] [   115  ]       [   12   ]  <-- Hash Function 1
Row 1   [   18   ] [   99   ] [    3   ]       [   42   ]  <-- Hash Function 2
Row 2   [   56   ] [   42   ] [   210  ]       [    1   ]  <-- Hash Function 3
...
Row d-1 [   42   ] [   14   ] [    8   ]       [   89   ]  <-- Hash Function d
```

### Error Guarantees

A Count-Min Sketch provides formal $(\epsilon, \delta)$ probabilistic guarantees:

$$\hat{a} \le a + \epsilon N$$

With probability at least $1 - \delta$. In this formulation, $a$ represents the true frequency, $\hat{a}$ is the estimated frequency, and $N$ is the cumulative stream volume. The error factor $\epsilon$ is controlled by matrix width $w = \lceil e / \epsilon \rceil$, while the failure probability $\delta$ is governed by depth $d = \lceil \ln(1 / \delta) \rceil$.

Because hash collisions only ever increment counters, **a Count-Min Sketch never underestimates frequency**. It only overestimates.

When querying the frequency of a key, the algorithm computes the hash index for each row and returns the **minimum value** across all rows:

$$\hat{a} = \min_{j \in [0, d-1]} \text{Table}[j, h_j(x)]$$

Taking the minimum discards rows affected by heavy hash collisions, isolating the counter closest to the true frequency.

### The Conservative Update Optimization

In standard updates, every row counter is incremented by one.

Under the **Conservative Update heuristic**, when adding key $x$:

1. First, query the current minimum count $\hat{a}_{current}$ across all rows for key $x$.
2. Increment strictly those counters whose current value equals $\hat{a}_{current}$. Counters that are already higher are left unchanged.

This simple heuristic reduces estimation error by up to 50% without altering the space complexity or breaking error bounds.

### Production Implementation

```python
import math
import mmh3

class CountMinSketch:
    def __init__(self, epsilon: float = 0.001, delta: float = 0.01):
        # Width w determines error margin; depth d determines confidence
        self.w = int(math.ceil(math.e / epsilon))
        self.d = int(math.ceil(math.log(1.0 / delta)))

        # Fixed size 2D integer array
        self.table = [[0] * self.w for _ in range(self.d)]

    def add(self, key: str, count: int = 1):
        # Conservative update: identify current minimum first
        current_min = self.estimate(key)
        target_value = current_min + count

        for row in range(self.d):
            col = mmh3.hash(key, seed=row, signed=False) % self.w
            if self.table[row][col] < target_value:
                self.table[row][col] = target_value

    def estimate(self, key: str) -> int:
        min_val = float("inf")
        for row in range(self.d):
            col = mmh3.hash(key, seed=row, signed=False) % self.w
            val = self.table[row][col]
            if val < min_val:
                min_val = val
        return int(min_val)
```

## 3. HyperLogLog: Cardinality Counting in 1.5 Kilobytes

Counting distinct elements (such as Daily Active Users or unique IPs hitting a firewall) classically requires maintaining a set of all seen keys. Storing 100 million UUIDs in a hash set consumes gigabytes of memory.

HyperLogLog (Flajolet et al., 2007) estimates distinct cardinality up to billions of unique items with a standard error of roughly 1% using strictly **1.5 kilobytes of memory**.

### The Mathematical Intuition: Leading Zeros

HyperLogLog relies on coin toss probability.

If you toss a fair coin, the probability of observing a sequence of $k$ consecutive heads is $2^{-k}$. If an observer tells you that their longest run of consecutive heads was 10, you can estimate that they probably tossed the coin roughly $2^{10} \approx 1024$ times.

HyperLogLog translates this to binary hashes:

1. A cryptographic or uniform hash function maps each input key to a random 64-bit integer.
2. The binary representation of a random hash behaves identically to independent coin flips.
3. The number of leading zeros in the hash indicates population size. If the maximum run of leading zeros seen across a stream is $z$, the cardinality estimate scales proportionally to $2^z$.

### Preventing Variance with Stochastic Averaging

Relying on a single register produces massive statistical variance. One lucky hash with 30 leading zeros would inflate the estimate into billions.

HyperLogLog eliminates this variance through **stochastic averaging**:

1. The first $p$ bits of the 64-bit hash select one of $m = 2^p$ independent registers (buckets). Standard implementations use $p = 12$ to $14$, allocating $m = 4096$ to $16384$ registers.
2. The remaining $64 - p$ bits are inspected for the number of leading zeros plus one ($q$).
3. The register stores the maximum $q$ observed for that bucket:

$$M[j] = \max(M[j], q)$$

To aggregate the registers, HyperLogLog computes the **harmonic mean** of $2^{M[j]}$, rather than the arithmetic mean. The harmonic mean strongly penalizes outliers, preventing rare long sequences from skewing the estimate:

$$E = \alpha_m \cdot m^2 \cdot \left( \sum_{j=1}^m 2^{-M[j]} \right)^{-1}$$

Where $\alpha_m$ is a bias correction constant ($0.7213 / (1 + 1.079 / m)$).

### HyperLogLog Register Sizing

Each register only needs to store values up to 64 (the maximum possible leading zeros in a 64-bit hash). Since $2^6 = 64$, **each register requires strictly 6 bits of storage**.

For $m = 16384$ registers ($p = 14$):

$$\text{Total Memory} = 16384 \times 6 \text{ bits} = 98304 \text{ bits} = 12288 \text{ bytes} \approx 12 \text{ KB}$$

With 12 kilobytes of memory, HyperLogLog counts cardinalities from zero to $10^{12}$ with a standard error of:

$$\text{Standard Error} = \frac{1.04}{\sqrt{m}} = \frac{1.04}{\sqrt{16384}} \approx 0.81\%$$

### Mergeability in Distributed Systems

HyperLogLog is an idempotent, commutative mergeable sketch.

If you have ten worker nodes processing incoming traffic across ten global regions, each node maintains its own local HLL register array. To compute global distinct users, you merge the registers by taking the element-wise maximum:

$$M_{global}[j] = \max(M_{region1}[j], M_{region2}[j], \dots, M_{region10}[j])$$

Merging requires zero data re-hashing and completes in sub millisecond CPU time. This makes HyperLogLog the standard cardinality engine inside distributed systems such as Redis (`PFADD` / `PFCOUNT`), Presto, BigQuery, and ClickHouse.

## Architectural Decision Matrix

| Data Structure | Core Question Answered | Space Complexity | Error Profile | Production Example |
| --- | --- | --- | --- | --- |
| Bloom Filter | Has this element possibly been seen? | $O(n)$ bits (approx 10 bits/item for 1% error) | False positives only (no false negatives) | RocksDB SSTable read bypass, Squid web cache |
| Count-Min Sketch | What is the approximate frequency of key X? | $O(w \cdot d)$ fixed memory | Overestimation only (no underestimation) | Network DDoS detection, streaming top-k heavy hitters |
| HyperLogLog | How many distinct elements exist in stream? | $O(m)$ fixed memory (12 KB for 0.8% error) | Symmetrical statistical variance | Redis PFCOUNT, database cardinality estimation |

Engineering maturity is recognizing that absolute accuracy is often an unnecessary luxury. When storage ceilings, network latency, and memory bandwidth dominate system bottlenecks, probabilistic approximations provide mathematical guarantees that keep high throughput distributed infrastructure fast and resilient.
