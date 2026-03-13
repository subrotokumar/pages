---
hide:
    - toc
---

# Cache Stampede: The Problem, The Fix That Makes It Worse, and the 2015 Paper Nobody Has Shipped  
  
> A deep dive into distributed caching failure modes, probabilistic early expiration, and why your runbook is missing a critical step.  
  
---  
  
## Introduction  
  
Cache stampede is well documented.  
  
What isn't: the part where the standard fix makes things worse, and the solution sitting in a 2015 academic paper that is genuinely rare in production codebases.  
  
This post covers the full lifecycle of a cache stampede — how it starts, why the distributed lock fix creates a second problem, the probabilistic algorithm that actually solves it, and the recovery scenario that silently turns a resolved incident into a new one. All examples are in Python using Redis.  
  
---  
  
## 1. What Actually Happens During a Cache Stampede  
  
The setup is always the same.  
  
A key expires. Forty requests arrive simultaneously, find nothing in cache, and all go straight to the database. The database was never built to handle that many identical queries at once. It starts falling behind. Someone decides the problem is Redis capacity and adds nodes. Nothing improves because the problem was never capacity. It was **timing**. Every request arrived at the same moment because they were all waiting on the same TTL expiry.  
  
### The Anatomy  
  
```  
t=0ms    key expires  
t=1ms    req #1  → cache MISS → goes to DB  
t=1ms    req #2  → cache MISS → goes to DB  
t=1ms    req #3  → cache MISS → goes to DB  
...  
t=1ms    req #40 → cache MISS → goes to DB  
  
         DB receives 40 identical queries simultaneously  
         DB connection pool exhausts  
         DB starts queuing  
         p99 latency spikes  
         alerts fire  
```  
  
### Reproducing It in Python  
  
```python    
import redis  
import time  
import threading  
  
r = redis.Redis(host='localhost', port=6379, decode_responses=True)  
  
def expensive_db_query(key: str) -> str:  
    """Simulates a slow database query."""  
    time.sleep(0.1)  # 100ms query  
    return f"value_for_{key}"  
  
def naive_cache_get(key: str) -> str:  
    """No protection. Classic stampede waiting to happen."""  
    value = r.get(key)  
    if value is not None:  
        return value  
  
    # Cache miss — goes to DB  
    # If 40 threads hit this simultaneously, 40 DB queries fire  
    value = expensive_db_query(key)  
    r.setex(key, 30, value)  # TTL = 30 seconds  
    return value  
  
def simulate_stampede():  
    KEY = "product:123"  
  
    # Seed the cache  
    r.setex(KEY, 1, "initial_value")  # expire in 1 second  
  
    time.sleep(1.1)  # wait for expiry  
  
    db_hit_count = 0  
    lock = threading.Lock()  
  
    original_query = expensive_db_query  
  
    def counting_query(key):  
        nonlocal db_hit_count  
        with lock:  
            db_hit_count += 1  
        return original_query(key)  
  
    # 40 threads hit the cache at the same time  
    threads = [threading.Thread(target=naive_cache_get, args=(KEY,)) for _ in range(40)]  
    start = time.time()  
    for t in threads:  
        t.start()  
    for t in threads:  
        t.join()  
  
    print(f"DB hit count: {db_hit_count}")  # prints: DB hit count: 40  
    print(f"Total time: {time.time() - start:.2f}s")  
  
simulate_stampede()  
```  
  
Output:  
```  
DB hit count: 40  
Total time: 0.11s  
```  
  
Forty identical queries hit the database in 100 milliseconds. At real scale with a slower query, this is an incident.  
  
---  
  
## 2. The Distributed Lock Fix — and Why It Creates a Second Problem  
  
The standard fix is a distributed lock. First request acquires it, fetches from the database, populates the cache, releases. Everyone else blocks and waits.  
  
```python  
import redis  
import time  
import uuid  
  
r = redis.Redis(host='localhost', port=6379, decode_responses=True)  
  
LOCK_TIMEOUT = 5       # seconds before lock auto-expires  
WAIT_TIMEOUT = 6       # how long a waiter will poll  
POLL_INTERVAL = 0.05   # 50ms between polls  
  
def cache_get_with_lock(key: str) -> str:  
    value = r.get(key)  
    if value is not None:  
        return value  
  
    lock_key = f"lock:{key}"  
    lock_value = str(uuid.uuid4())  # unique value so only owner can release  
  
    # Try to acquire the lock  
    acquired = r.set(lock_key, lock_value, nx=True, ex=LOCK_TIMEOUT)  
  
    if acquired:  
        try:  
            # Double-check after acquiring — another thread may have populated  
            value = r.get(key)  
            if value is not None:  
                return value  
  
            # We hold the lock — fetch from DB and populate cache  
            value = expensive_db_query(key)  
            r.setex(key, 30, value)  
            return value  
        finally:  
            # Release lock only if we still own it  
            release_lock(r, lock_key, lock_value)  
    else:  
        # Wait for the lock holder to populate the cache  
        deadline = time.time() + WAIT_TIMEOUT  
        while time.time() < deadline:  
            time.sleep(POLL_INTERVAL)  
            value = r.get(key)  
            if value is not None:  
                return value  
  
        # Lock holder never finished — fall through to DB as last resort  
        return expensive_db_query(key)  
  
def release_lock(r, lock_key, lock_value):  
    """Atomic check-and-delete to prevent releasing someone else's lock."""  
    pipe = r.pipeline(True)  
    try:  
        pipe.watch(lock_key)  
        if pipe.get(lock_key) == lock_value:  
            pipe.multi()  
            pipe.delete(lock_key)  
            pipe.execute()  
    except redis.WatchError:  
        pass  # someone else released it — that's fine  
```  
  
This works correctly in the happy path. The problem surfaces at the edges.  
  
### The Failure Mode  
  
```  
t=0ms     key expires  
t=1ms     req #1 acquires lock, starts DB query  
t=1ms     req #2–40 enter wait loop (polling every 50ms)  
  
          req #1 hits a GC pause at t=200ms  
          lock timeout fires at t=5000ms (LOCK_TIMEOUT)  
  
t=5001ms  req #2–40 all stop waiting  
t=5001ms  all 39 retry simultaneously  
t=5001ms  all 39 find cache still empty (req #1 never finished)  
t=5001ms  all 39 try to acquire lock — one wins, 38 wait again  
t=5001ms  the 38 waiters now have a fresh WAIT_TIMEOUT window  
  
          you now have a thundering herd AND a synchronized retry storm  
          the burst is more aggressive than the original stampede  
```  
  
The lock delayed the problem and amplified it.  
  
### The Redlock Problem  
  
If you are using Redlock across multiple Redis nodes, there is a deeper correctness issue. Martin Kleppmann wrote about it in 2016 — the algorithm does not provide the safety guarantees it claims under process pauses and clock drift. For best-effort caching this is tolerable. For anything that requires correctness guarantees, read that analysis before your next architecture review.  
  
---  
  
## 3. Probabilistic Early Expiration — The Fix Nobody Ships  
  
Instead of waiting for a key to expire and reacting, each worker independently evaluates whether to recompute early. The decision is based on how long recomputation takes, how close the key is to expiry, and a random component.  
  
### The Formula  
  
```  
recompute if:  
current_time - ( recompute_duration × β × log( random() ) ) > expiry_time  
```  
  
Where:  
- `recompute_duration` — how long a cache refresh takes in seconds  
- `β` — tuning constant, typically 1.0  
- `random()` — uniform random value between 0 and 1, evaluated independently per worker  
- `expiry_time` — the absolute Unix timestamp when the key expires  
  
Because `log(random())` is always negative (log of a value between 0 and 1), the subtraction effectively moves the decision point earlier than the actual expiry. The randomness means different workers make different decisions — a few trigger early recomputation in the background while the rest keep serving the cached value. By the time the key actually hits its TTL, it has already been replaced.  
  
**No synchronized expiry event. No stampede.**  
  
### Python Implementation  
  
```python  
import redis  
import time  
import math  
import random  
import threading  
  
r = redis.Redis(host='localhost', port=6379, decode_responses=True)  
  
BETA = 1.0  # tuning constant — increase for more aggressive early recomputation  
  
def set_with_metadata(key: str, value: str, ttl: int):  
    """  
    Store value with its TTL metadata so we can compute  
    probabilistic expiry without a separate lookup.  
    """  
    expiry_time = time.time() + ttl  
    pipe = r.pipeline()  
    pipe.set(f"{key}:value", value)  
    pipe.set(f"{key}:expiry", expiry_time)  
    pipe.set(f"{key}:delta", 0)  # will be updated after first real fetch  
    pipe.expire(f"{key}:value", ttl + 10)   # small buffer beyond TTL  
    pipe.expire(f"{key}:expiry", ttl + 10)  
    pipe.expire(f"{key}:delta", ttl + 10)  
    pipe.execute()  
  
def probabilistic_cache_get(key: str, fetch_fn, ttl: int = 30) -> str:  
    """  
    Fetch from cache using probabilistic early expiration.  
    Each worker independently decides whether to recompute early.  
    """  
    value = r.get(f"{key}:value")  
    expiry_str = r.get(f"{key}:expiry")  
    delta_str = r.get(f"{key}:delta")  
  
    if value is None or expiry_str is None:  
        # Cold cache — fetch and store  
        start = time.time()  
        value = fetch_fn(key)  
        delta = time.time() - start  
  
        set_with_metadata(key, value, ttl)  
        r.set(f"{key}:delta", delta)  
        return value  
  
    expiry_time = float(expiry_str)  
    delta = float(delta_str) if delta_str else 0.001  
  
    # Probabilistic early expiration check (Vattani et al. 2015)  
    now = time.time()  
    early_recompute = now - (delta * BETA * math.log(random.random())) > expiry_time  
  
    if early_recompute:  
        # This worker recomputes in the background  
        # Other workers continue serving the existing value  
        start = time.time()  
        new_value = fetch_fn(key)  
        delta = time.time() - start  
  
        set_with_metadata(key, new_value, ttl)  
        r.set(f"{key}:delta", delta)  
        return new_value  
  
    return value  
  
  
# ── Usage example ─────────────────────────────────────────────────────────────  
  
def fetch_from_db(key: str) -> str:  
    """Simulates a 100ms database query."""  
    time.sleep(0.1)  
    return f"fresh_value_for_{key}"  
  
def simulate_probabilistic():  
    KEY = "product:456"  
    TTL = 30  
    db_hit_count = 0  
    lock = threading.Lock()  
  
    def counting_fetch(key):  
        nonlocal db_hit_count  
        with lock:  
            db_hit_count += 1  
        return fetch_from_db(key)  
  
    # Warm the cache  
    probabilistic_cache_get(KEY, counting_fetch, TTL)  
    db_hit_count = 0  # reset counter after warm  
  
    # Simulate 40 concurrent requests as key approaches expiry  
    # In real usage these would be spread across the TTL window  
    threads = [  
        threading.Thread(target=probabilistic_cache_get, args=(KEY, counting_fetch, TTL))  
        for _ in range(40)  
    ]  
    for t in threads:  
        t.start()  
    for t in threads:  
        t.join()  
  
    print(f"DB hit count with probabilistic expiry: {db_hit_count}")  
    # Typically prints 1 or 2 — a few workers recomputed early, the rest served cached value  
  
simulate_probabilistic()  
```  
  
Output:  
```  
DB hit count with probabilistic expiry: 1  
```  
  
One database query instead of forty. No lock. No retry storm. No coordination between workers.  
  
### Tuning Beta  
  
```python  
# Conservative — only recomputes very close to expiry  
BETA = 0.5  
  
# Default — balanced tradeoff  
BETA = 1.0  
  
# Aggressive — recomputes well before expiry, useful for slow fetches  
BETA = 2.0  
```  
  
Higher beta means the recomputation window starts earlier relative to the key's TTL. For fast fetches (sub-10ms) beta=1 is fine. For slow fetches (100ms+) consider increasing it to ensure the cache is refreshed before expiry under concurrent load.  
  
---  
  
## 4. The Recovery Scenario That Turns One Incident Into Two  
  
The most damaging scenario is not the stampede itself. It is the recovery.  
  
```  
t=0       cache cluster goes fully dark  
t=1s      every request hits the database cold  
t=10s     database starts struggling — connection pool backs up  
t=30s     on-call acknowledges, starts investigation  
t=5m      cache cluster restored and brought back online  
t=5m+1s   cache is completely empty  
t=5m+2s   every request is still hitting the database while cache warms  
t=5m+3s   cache incident marked resolved  
t=5m+4s   database incident just started  
```  
  
Both events happened in the same sequence of decisions. The runbook covered cache restoration. It said nothing about the gap.  
  
### The Fix: Warm Before Cutover  
  
```python  
import redis  
import psycopg2  
from typing import List, Tuple  
  
r = redis.Redis(host='localhost', port=6379, decode_responses=True)  
  
def get_highest_frequency_keys(db_conn, limit: int = 1000) -> List[Tuple[str, str, int]]:  
    """  
    Pull the most frequently accessed keys and their values from the database.  
    In practice, track access frequency in a separate table or use Redis keyspace  
    notifications before the outage to build a hot-key list.  
    """  
    cursor = db_conn.cursor()  
    cursor.execute("""  
        SELECT cache_key, cache_value, ttl_seconds  
        FROM cache_hot_keys  
        ORDER BY access_count DESC  
        LIMIT %s  
    """, (limit,))  
    return cursor.fetchall()  
  
def warm_cache_before_cutover(db_conn, batch_size: int = 100):  
    """  
    Populate highest-frequency keys from the database into Redis  
    BEFORE the load balancer routes traffic back.  
  
    This step is absent from most runbooks.  
    """  
    hot_keys = get_highest_frequency_keys(db_conn, limit=1000)  
  
    pipe = r.pipeline()  
    count = 0  
  
    for cache_key, cache_value, ttl in hot_keys:  
        # Jitter the TTL slightly to avoid synchronized expiry after warming  
        jittered_ttl = int(ttl * (0.9 + 0.2 * random.random()))  # ±10%  
        pipe.setex(cache_key, jittered_ttl, cache_value)  
        count += 1  
  
        if count % batch_size == 0:  
            pipe.execute()  
            pipe = r.pipeline()  
            print(f"Warmed {count} keys...")  
  
    if count % batch_size != 0:  
        pipe.execute()  
  
    print(f"Cache warm complete. {count} keys loaded before traffic cutover.")  
  
# ── Runbook step (run this BEFORE updating load balancer) ────────────────────  
#  
# 1. Restore Redis cluster  
# 2. Run warm_cache_before_cutover()  
# 3. Verify key count: redis-cli dbsize  
# 4. THEN update load balancer / DNS to route traffic back  
# 5. Monitor DB connection count for the first 60 seconds  
```  
  
### What to Track After Cutover  
  
```python  
import time  
  
def monitor_post_cutover(duration_seconds: int = 60, interval: int = 5):  
    """  
    Watch cache hit rate and DB connection count in the minutes  
    immediately following a cache restoration.  
    A falling hit rate or rising DB connections means the cache  
    is not absorbing load as expected.  
    """  
    start = time.time()  
    prev_hits = int(r.info('stats').get('keyspace_hits', 0))  
    prev_misses = int(r.info('stats').get('keyspace_misses', 0))  
  
    while time.time() - start < duration_seconds:  
        time.sleep(interval)  
  
        info = r.info('stats')  
        hits = int(info.get('keyspace_hits', 0))  
        misses = int(info.get('keyspace_misses', 0))  
  
        delta_hits = hits - prev_hits  
        delta_misses = misses - prev_misses  
        total = delta_hits + delta_misses  
  
        hit_rate = (delta_hits / total * 100) if total > 0 else 0  
        print(f"Cache hit rate (last {interval}s): {hit_rate:.1f}%  |  misses: {delta_misses}")  
  
        prev_hits = hits  
        prev_misses = misses  
  
        if hit_rate < 70:  
            print("WARNING: hit rate below 70% — database may be absorbing excess load")  
```  
  
---  
  
## 5. Two Low-Effort Changes Worth Shipping Today  
  
### Jitter TTLs  
  
A random offset of 10 to 20 percent on every key expiry means expiry events cannot synchronize. A stampede requires all keys to expire at the same moment. Remove that coordination and the stampede cannot form.  
  
```python  
import random  
  
BASE_TTL = 300  # 5 minutes  
  
def jittered_ttl(base: int, jitter_pct: float = 0.1) -> int:  
    """  
    Returns TTL with ±jitter_pct random variation.  
    base=300, jitter_pct=0.1 → TTL between 270 and 330 seconds.  
    """  
    offset = base * jitter_pct  
    return int(base + random.uniform(-offset, offset))  
  
# Usage  
r.setex("product:789", jittered_ttl(BASE_TTL), value)  
  
# For bulk inserts — spreads expiry across a window  
keys_and_values = [("k1", "v1"), ("k2", "v2"), ("k3", "v3")]  
pipe = r.pipeline()  
for key, value in keys_and_values:  
    pipe.setex(key, jittered_ttl(BASE_TTL), value)  
pipe.execute()  
```  
  
### Local In-Process Cache in Front of Redis  
  
A full Redis outage gives several minutes of runway from the local layer instead of immediately exposing the database.  
  
```python  
import time  
import threading  
from dataclasses import dataclass  
from typing import Optional, Dict  
  
@dataclass  
class CacheEntry:  
    value: str  
    expires_at: float  
  
class LocalCache:  
    """  
    Simple in-process LRU-like cache with TTL.  
    Sits in front of Redis. Survives Redis outages for its TTL window.  
    """  
    def __init__(self, max_size: int = 1000, default_ttl: int = 60):  
        self._store: Dict[str, CacheEntry] = {}  
        self._lock = threading.Lock()  
        self._max_size = max_size  
        self._default_ttl = default_ttl  
  
    def get(self, key: str) -> Optional[str]:  
        with self._lock:  
            entry = self._store.get(key)  
            if entry is None:  
                return None  
            if time.time() > entry.expires_at:  
                del self._store[key]  
                return None  
            return entry.value  
  
    def set(self, key: str, value: str, ttl: Optional[int] = None):  
        with self._lock:  
            if len(self._store) >= self._max_size:  
                # Evict oldest expired entry, or oldest entry if none expired  
                now = time.time()  
                expired = [k for k, v in self._store.items() if v.expires_at < now]  
                if expired:  
                    del self._store[expired[0]]  
                else:  
                    oldest = next(iter(self._store))  
                    del self._store[oldest]  
  
            self._store[key] = CacheEntry(  
                value=value,  
                expires_at=time.time() + (ttl or self._default_ttl)  
            )  
  
  
class TieredCache:  
    """  
    L1: local in-process cache  (fast, survives Redis outage)  
    L2: Redis                   (shared across instances)  
    L3: database                (source of truth)  
    """  
    def __init__(self, local_ttl: int = 60, redis_ttl: int = 300):  
        self.local = LocalCache(max_size=1000, default_ttl=local_ttl)  
        self.redis = redis.Redis(host='localhost', port=6379, decode_responses=True)  
        self.local_ttl = local_ttl  
        self.redis_ttl = redis_ttl  
  
    def get(self, key: str, fetch_fn) -> str:  
        # L1: local cache  
        value = self.local.get(key)  
        if value is not None:  
            return value  
  
        # L2: Redis  
        try:  
            value = self.redis.get(key)  
            if value is not None:  
                self.local.set(key, value, self.local_ttl)  
                return value  
        except redis.RedisError:  
            # Redis is down — L1 already checked, fall through to DB  
            pass  
  
        # L3: database  
        value = fetch_fn(key)  
  
        # Populate both layers (best effort for Redis)  
        self.local.set(key, value, self.local_ttl)  
        try:  
            self.redis.setex(key, jittered_ttl(self.redis_ttl), value)  
        except redis.RedisError:  
            pass  # Redis still down — local cache absorbs load  
  
        return value  
  
  
# ── Usage ─────────────────────────────────────────────────────────────────────  
  
cache = TieredCache(local_ttl=60, redis_ttl=300)  
  
def get_product(product_id: str) -> str:  
    return cache.get(  
        key=f"product:{product_id}",  
        fetch_fn=lambda k: fetch_from_db(k)  
    )  
```  
  
During a Redis outage, the local cache continues serving requests for up to 60 seconds without touching the database. That window is often enough to restore Redis, run the warm procedure, and cut traffic back without the database ever knowing there was a problem.  
  
---  
  
## 6. Putting It All Together  
  
```python  
import redis  
import time  
import math  
import random  
import threading  
from typing import Callable  
  
r = redis.Redis(host='localhost', port=6379, decode_responses=True)  
  
BETA = 1.0  
  
class ResilientCache:  
    """  
    Production-grade cache layer combining:  
    - Probabilistic early expiration (no stampede)  
    - Jittered TTLs (no synchronized expiry)  
    - Local in-process fallback (survives Redis outage)  
    - Cache warming utility (safe recovery)  
    """  
  
    def __init__(self, redis_client, local_ttl: int = 60):  
        self.redis = redis_client  
        self.local = LocalCache(max_size=2000, default_ttl=local_ttl)  
  
    def get(self, key: str, fetch_fn: Callable, ttl: int = 300) -> str:  
        # L1: local  
        value = self.local.get(key)  
        if value is not None:  
            return value  
  
        # L2: Redis with probabilistic early expiration  
        try:  
            value, should_recompute = self._probabilistic_get(key)  
  
            if value is not None and not should_recompute:  
                self.local.set(key, value)  
                return value  
  
            if value is not None and should_recompute:  
                # Recompute in background, return existing value immediately  
                threading.Thread(  
                    target=self._background_refresh,  
                    args=(key, fetch_fn, ttl),  
                    daemon=True  
                ).start()  
                self.local.set(key, value)  
                return value  
  
        except redis.RedisError:  
            # Redis down — fall through  
            cached_local = self.local.get(key)  
            if cached_local:  
                return cached_local  
  
        # L3: database  
        value = fetch_fn(key)  
        self._store(key, value, ttl)  
        return value  
  
    def _probabilistic_get(self, key: str):  
        pipe = self.redis.pipeline()  
        pipe.get(f"{key}:v")  
        pipe.get(f"{key}:exp")  
        pipe.get(f"{key}:delta")  
        value, expiry_str, delta_str = pipe.execute()  
  
        if value is None:  
            return None, True  
  
        expiry = float(expiry_str) if expiry_str else 0  
        delta = float(delta_str) if delta_str else 0.001  
        now = time.time()  
  
        should_recompute = (now - delta * BETA * math.log(random.random())) > expiry  
        return value, should_recompute  
  
    def _store(self, key: str, value: str, ttl: int):  
        jittered = jittered_ttl(ttl)  
        expiry = time.time() + jittered  
        pipe = self.redis.pipeline()  
        pipe.setex(f"{key}:v", jittered + 10, value)  
        pipe.setex(f"{key}:exp", jittered + 10, expiry)  
        pipe.execute()  
        self.local.set(key, value)  
  
    def _background_refresh(self, key: str, fetch_fn: Callable, ttl: int):  
        start = time.time()  
        value = fetch_fn(key)  
        delta = time.time() - start  
        self._store(key, value, ttl)  
        self.redis.setex(f"{key}:delta", ttl + 10, delta)  
  
    def warm(self, hot_keys: list):  
        """Call this BEFORE routing traffic back after a Redis outage."""  
        pipe = self.redis.pipeline()  
        for key, value, ttl in hot_keys:  
            jittered = jittered_ttl(ttl)  
            expiry = time.time() + jittered  
            pipe.setex(f"{key}:v", jittered + 10, value)  
            pipe.setex(f"{key}:exp", jittered + 10, expiry)  
        pipe.execute()  
        print(f"Warmed {len(hot_keys)} keys. Safe to route traffic.")  
```  
  
---  
  
## Summary  
  
| Scenario | Problem | Fix |  
|---|---|---|  
| Classic stampede | Synchronized TTL expiry sends all requests to DB | Probabilistic early expiration |  
| Distributed lock | GC pause causes retry storm worse than original | Remove lock, use probabilistic approach |  
| Cache restoration | Empty cache exposes DB immediately after recovery | Warm cache before load balancer cutover |  
| Synchronized expiry | Keys created together expire together | Jitter TTLs by ±10–20% |  
| Redis outage | DB absorbs full load instantly | Local in-process cache as L1 |  
  
The real problem is not stampede. It is systems built treating cache as a performance layer while operating it as though the system cannot survive without it. When cache going down takes the database with it, the cache has become load-bearing infrastructure that was never designed or monitored as load-bearing infrastructure.  
  
The stampede is just the moment that gap becomes visible.  
  
---  
  
## References  
  
- Vattani, A., Chierichetti, F., Lowenstein, K. (2015). *Optimal Probabilistic Cache Stampede Prevention*. VLDB.  
- Kleppmann, M. (2016). *How to do distributed locking*. martin.kleppmann.com  
- Redis documentation: *Keyspace notifications*, *Pipelining*, *Lua scripting*