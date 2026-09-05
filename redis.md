# Redis Complete Guide (Docker Edition)

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Redis](#introducing-redis)
2. [Installation & Setup with Docker](#installation--setup-with-docker)
3. [Redis CLI Basics](#redis-cli-basics)
4. [Data Types & Commands](#data-types--commands)
5. [Key Management](#key-management)
6. [Pub/Sub Messaging](#pubsub-messaging)
7. [Transactions](#transactions)
8. [Lua Scripting](#lua-scripting)
9. [Persistence](#persistence)
10. [Replication](#replication)
11. [Redis Sentinel](#redis-sentinel)
12. [Redis Cluster](#redis-cluster)
13. [Security](#security)
14. [Performance & Optimization](#performance--optimization)

---

## Introducing Redis

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store used as a database, cache, message broker, and streaming engine. It stores data in memory for extremely fast reads and writes, typically achieving sub-millisecond response times. Unlike traditional databases that store data on disk, Redis keeps the entire dataset in RAM and optionally persists it to disk for durability.

Redis is commonly used for caching frequently accessed data to reduce database load, session management in web applications, real-time leaderboards and counters, rate limiting, message queues and pub/sub messaging, and as a primary database for use cases where speed matters most.

Core concepts of Redis are:

- **Key-Value Store**: Every piece of data is stored as a key-value pair. Keys are strings, values can be various data types.
- **In-Memory**: All data lives in RAM, making operations extremely fast (100,000+ operations per second).
- **Single-Threaded**: Redis processes commands sequentially in a single thread, eliminating race conditions and the need for locks.
- **Persistence**: Optional disk persistence through RDB snapshots and AOF (Append Only File) logs.
- **Data Structures**: Unlike simple key-value stores, Redis supports strings, lists, sets, sorted sets, hashes, streams, and more.
- **Atomic Operations**: All Redis commands are atomic, meaning they either fully complete or do not execute at all.

---

## Installation & Setup with Docker

Throughout this guide, every Redis instance runs inside Docker. No host installation is needed.

### Pull and Run Redis

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

`-d` runs the container in the background. `--name redis` gives it a readable name. `-p 6379:6379` maps the default Redis port so the host can connect. `redis:7` pulls the official Redis 7 image.

### Verify It's Running

```bash
docker exec -it redis redis-cli ping
# PONG
```

`docker exec -it redis redis-cli` opens an interactive Redis CLI session inside the running container. `ping` is a health check command that returns `PONG` if the server is reachable.

### Check Server Version

```bash
docker exec -it redis redis-server --version
# Redis server v=7.x.x sha=00000000:0 malloc=jemalloc-5.x.x bits=64
```

### Run on a Custom Port

```bash
docker run -d --name redis -p 6380:6380 redis:7 redis-server --port 6380
```

`--port 6380` tells Redis to listen on port 6380 inside the container. `-p 6380:6380` maps that container port to the same port on the host. To connect: `docker exec -it redis redis-cli -p 6380`.

### Run with Password

```bash
docker run -d --name redis -p 6379:6379 redis:7 redis-server --requirepass yourpassword
```

This overrides the default container command and starts Redis with password authentication.

### Run with Persistent Storage

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7 redis-server --appendonly yes
```

`-v redis-data:/data` mounts a named volume to `/data`, where Redis stores its dump files. `--appendonly yes` enables AOF persistence so data survives container restarts.

### Run with Custom Config

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  -v ./redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7 redis-server /usr/local/etc/redis/redis.conf
```

This bind-mounts a custom `redis.conf` from the host into the container and tells Redis to use it.

### Docker Compose: FastAPI + Redis

This is the Docker Compose setup used throughout this guide. Every FastAPI example below assumes this environment.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

`depends_on` ensures Redis starts before the application. The `api` service connects to Redis using `redis` as the hostname (the service name) because Docker Compose creates a shared network.

The FastAPI app uses this `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

And this `requirements.txt`:

```
fastapi
uvicorn
redis
```

Start everything with:

```bash
docker compose up --build -d
```

### FastAPI Example: Health Check

```python
# main.py
import os
import redis
from fastapi import FastAPI

app = FastAPI()

r = redis.from_url(os.getenv("REDIS_URL", "redis://localhost:6379/0"))

@app.get("/health")
def health():
    return {"redis_ping": r.ping()}  # → {"redis_ping": true}
```

`redis.from_url()` connects to Redis using the URL from the environment variable. `r.ping()` returns `True` if the server responds, making it a simple health check.

---

## Redis CLI Basics

`redis-cli` is the command-line interface for interacting with Redis. Inside Docker, you access it via `docker exec`.

### Connecting

```bash
docker exec -it redis redis-cli                       # Connect to Redis inside container
docker exec -it redis redis-cli -a yourpassword       # Connect with password

# From host (if redis-cli is installed locally)
redis-cli -h 127.0.0.1 -p 6379
redis-cli -h 127.0.0.1 -p 6379 -a yourpassword
```

`-h` specifies the hostname. `-p` specifies the port. `-a` provides the authentication password.

### Useful CLI Commands

```bash
docker exec -it redis redis-cli INFO                  # Server info and statistics
docker exec -it redis redis-cli INFO memory           # Memory usage details
docker exec -it redis redis-cli DBSIZE                # Number of keys in current database
docker exec -it redis redis-cli MONITOR               # Watch all commands in real time
docker exec -it redis redis-cli CONFIG GET maxmemory  # Get a config value
docker exec -it redis redis-cli SLOWLOG GET 10        # Show 10 slowest queries
docker exec -it redis redis-cli CLIENT LIST           # List all connected clients
```

`MONITOR` streams every command Redis receives in real time, useful for debugging but expensive in production. `SLOWLOG` shows commands that exceeded the configured slow threshold (default 10ms).

### Select Database

```bash
SELECT 0                               # Switch to database 0 (default)
SELECT 3                               # Switch to database 3
```

Redis provides 16 databases (0-15) by default. Each database has its own keyspace. `SELECT` switches the current connection to the specified database. In production, most applications use database 0.

### Flush Data

```bash
FLUSHDB                                # Delete all keys in current database
FLUSHALL                               # Delete all keys in ALL databases
```

`FLUSHDB` clears only the currently selected database. `FLUSHALL` wipes every database. Both commands are dangerous in production.

### FastAPI Example: Server Info

```python
@app.get("/redis/info")
def redis_info():
    info = r.info()
    return {
        "redis_version": info["redis_version"],
        "connected_clients": info["connected_clients"],
        "used_memory_human": info["used_memory_human"],
        "total_keys": r.dbsize(),
    }
```

`r.info()` returns a dictionary of server stats. `r.dbsize()` returns the number of keys in the current database.

---

## Data Types & Commands

Redis supports multiple data types, each optimized for different use cases. Every value stored in Redis is associated with a string key.

### Strings

Strings are the simplest data type. A string value can hold text, numbers, or binary data up to 512 MB.

```bash
SET name "Bidur"                       # Set a key
GET name                               # Get value → "Bidur"
SET counter 100                        # Strings can hold numbers
INCR counter                           # Increment by 1 → 101
INCRBY counter 10                      # Increment by 10 → 111
DECR counter                           # Decrement by 1 → 110
DECRBY counter 5                       # Decrement by 5 → 105
INCRBYFLOAT price 2.5                  # Increment by float

APPEND name " Sapkota"                 # Append to string → "Bidur Sapkota"
STRLEN name                            # String length → 14

SETNX lock "acquired"                  # Set only if key does NOT exist
SETEX session 3600 "data"              # Set with expiration (3600 seconds)
PSETEX session 5000 "data"             # Set with expiration in milliseconds

MSET k1 "v1" k2 "v2" k3 "v3"         # Set multiple keys at once
MGET k1 k2 k3                         # Get multiple values at once

GETSET counter 0                       # Get old value and set new one
GETRANGE name 0 4                      # Get substring → "Bidur"
SETRANGE name 6 "S."                   # Overwrite part of string
```

`SET` creates or overwrites a key. `INCR`/`DECR` treat the string as an integer and perform atomic increment/decrement; they fail if the value is not a valid integer. `SETNX` (Set if Not eXists) is useful for implementing distributed locks. `SETEX` combines `SET` and `EXPIRE` into a single atomic operation. `MSET`/`MGET` operate on multiple keys in a single command, reducing round trips. `GETSET` atomically sets a new value and returns the old one.

### SET Options

```bash
SET key "value" EX 60                  # Expire in 60 seconds
SET key "value" PX 5000                # Expire in 5000 milliseconds
SET key "value" NX                     # Set only if key does not exist
SET key "value" XX                     # Set only if key already exists
SET key "value" GET                    # Return old value while setting new
SET key "value" EX 60 NX              # Combine: expire + only if not exists
```

Modern Redis versions allow passing options directly to `SET` instead of using `SETNX`, `SETEX`, etc. `EX` sets seconds-based expiry. `PX` sets milliseconds-based expiry. `NX` and `XX` control conditional setting. `GET` returns the previous value.

### FastAPI Example: Strings (Page View Counter)

```python
@app.get("/page/{page_name}")
def view_page(page_name: str):
    count = r.incr(f"pageviews:{page_name}")
    return {"page": page_name, "views": count}

@app.get("/page/{page_name}/stats")
def page_stats(page_name: str):
    count = r.get(f"pageviews:{page_name}")
    return {"page": page_name, "views": int(count) if count else 0}
```

`r.incr()` atomically increments and returns the new count. Each page visit bumps the counter. `r.get()` retrieves the current value.

---

### Lists

Lists are ordered collections of strings, implemented as linked lists. They allow fast insertions at head or tail and are ideal for queues, stacks, and timelines.

```bash
LPUSH tasks "task1"                    # Push to head (left)
LPUSH tasks "task2" "task3"            # Push multiple to head
RPUSH tasks "task4"                    # Push to tail (right)

LRANGE tasks 0 -1                      # Get all elements
LRANGE tasks 0 2                       # Get first 3 elements (0-indexed)

LPOP tasks                             # Remove and return from head
RPOP tasks                             # Remove and return from tail
BLPOP tasks 30                         # Blocking pop (wait up to 30 seconds)
BRPOP tasks 30                         # Blocking pop from tail

LLEN tasks                             # List length
LINDEX tasks 0                         # Get element at index 0
LSET tasks 0 "updated"                 # Set element at index 0
LINSERT tasks BEFORE "task4" "task3.5" # Insert before a specific element
LREM tasks 2 "task1"                   # Remove 2 occurrences of "task1"
LTRIM tasks 0 99                       # Keep only elements at index 0-99

RPOPLPUSH source destination           # Pop from source tail, push to dest head
LMOVE source dest LEFT RIGHT           # Move element between lists
```

`LPUSH` adds to the head (left end) of the list; multiple values are pushed one by one from left to right, so `LPUSH tasks "a" "b" "c"` results in `["c", "b", "a"]`. `LRANGE 0 -1` returns all elements; `-1` refers to the last element. `BLPOP`/`BRPOP` block the connection until an element is available or the timeout expires, making them ideal for worker queues. `LTRIM` is often used after `LPUSH` to keep a list capped at a fixed length. `RPOPLPUSH` atomically moves an element from one list to another, useful for reliable queue processing.

### FastAPI Example: Lists (Task Queue)

```python
from pydantic import BaseModel

class Task(BaseModel):
    name: str

@app.post("/tasks")
def add_task(task: Task):
    r.rpush("tasks", task.name)
    length = r.llen("tasks")
    return {"added": task.name, "queue_length": length}

@app.get("/tasks")
def get_tasks():
    tasks = r.lrange("tasks", 0, -1)
    return {"tasks": [t.decode() for t in tasks]}

@app.post("/tasks/next")
def process_next_task():
    task = r.lpop("tasks")
    if task is None:
        return {"message": "queue is empty"}
    return {"processing": task.decode()}
```

`r.rpush()` adds to the tail of the list (FIFO queue). `r.lrange(0, -1)` returns all elements. `r.lpop()` pops from the head, processing the oldest task first.

---

### Sets

Sets are unordered collections of unique strings. They support fast membership testing, intersection, union, and difference operations.

```bash
SADD tags "redis" "docker" "python"    # Add members
SMEMBERS tags                          # Get all members
SCARD tags                             # Count members → 3
SISMEMBER tags "redis"                 # Check membership → 1 (true)
SMISMEMBER tags "redis" "go" "python"  # Check multiple → [1, 0, 1]

SREM tags "python"                     # Remove a member
SPOP tags                              # Remove and return a random member
SRANDMEMBER tags 2                     # Return 2 random members (no removal)

# Set operations
SADD frontend "html" "css" "js" "react"
SADD backend "python" "js" "redis" "docker"

SINTER frontend backend                # Intersection → {"js"}
SUNION frontend backend                # Union → all unique members
SDIFF frontend backend                 # In frontend but NOT in backend

SINTERSTORE result frontend backend    # Store intersection in "result"
SUNIONSTORE result frontend backend    # Store union in "result"
SDIFFSTORE result frontend backend     # Store difference in "result"

SMOVE frontend backend "react"         # Move member between sets
```

`SADD` adds members to the set; duplicates are ignored. `SISMEMBER` returns `1` if the member exists, `0` otherwise. `SINTER` returns members common to all given sets. `SDIFF` returns members in the first set that are not in any of the subsequent sets. The `STORE` variants save the result to a new key instead of returning it.

### FastAPI Example: Sets (Tag System)

```python
@app.post("/articles/{article_id}/tags/{tag}")
def add_tag(article_id: int, tag: str):
    r.sadd(f"article:{article_id}:tags", tag)
    return {"article_id": article_id, "tag_added": tag}

@app.get("/articles/{article_id}/tags")
def get_tags(article_id: int):
    tags = r.smembers(f"article:{article_id}:tags")
    return {"article_id": article_id, "tags": [t.decode() for t in tags]}

@app.get("/articles/common-tags")
def common_tags(id1: int, id2: int):
    common = r.sinter(f"article:{id1}:tags", f"article:{id2}:tags")
    return {"common_tags": [t.decode() for t in common]}
```

`r.sadd()` adds a tag; duplicates are ignored automatically. `r.smembers()` returns all tags. `r.sinter()` finds tags common to two articles.

---

### Sorted Sets (ZSets)

Sorted sets are like sets, but each member has an associated score (float). Members are unique, but scores can repeat. Members are always ordered by score (ascending).

```bash
ZADD leaderboard 100 "alice"           # Add with score
ZADD leaderboard 85 "bob" 92 "carol"  # Add multiple members
ZADD leaderboard GT 110 "alice"        # Update only if new score is greater

ZRANGE leaderboard 0 -1               # Get all members (low to high)
ZRANGE leaderboard 0 -1 WITHSCORES    # Include scores in output
ZREVRANGE leaderboard 0 2             # Top 3 (high to low)
ZRANGEBYSCORE leaderboard 80 100      # Members with score 80-100

ZSCORE leaderboard "alice"             # Get score → 100
ZRANK leaderboard "alice"              # Rank (0-indexed, low to high)
ZREVRANK leaderboard "alice"           # Rank (0-indexed, high to low)
ZCARD leaderboard                      # Count members

ZINCRBY leaderboard 5 "bob"            # Increment score by 5
ZREM leaderboard "carol"               # Remove a member
ZREMRANGEBYSCORE leaderboard 0 50      # Remove members with score 0-50
ZREMRANGEBYRANK leaderboard 0 1        # Remove members at rank 0-1

ZCOUNT leaderboard 80 100              # Count members with score 80-100

# Set operations on sorted sets
ZINTERSTORE result 2 set1 set2         # Intersection with summed scores
ZUNIONSTORE result 2 set1 set2 WEIGHTS 2 1  # Union with weighted scores
```

`ZADD` adds members with scores; if a member already exists, its score is updated. `GT` only updates if the new score is greater than the current one (useful for leaderboards). `ZRANGE 0 -1 WITHSCORES` returns all members with their scores. `ZREVRANGE` returns members sorted from highest to lowest score. `ZRANK` returns the 0-based position; the member with the lowest score has rank 0. `ZINCRBY` atomically increments a member's score. `ZINTERSTORE`/`ZUNIONSTORE` combine sorted sets with optional `WEIGHTS` that multiply scores.

### FastAPI Example: Sorted Sets (Leaderboard)

```python
@app.post("/leaderboard/{player}")
def add_score(player: str, score: float):
    r.zadd("leaderboard", {player: score})
    rank = r.zrevrank("leaderboard", player)
    return {"player": player, "score": score, "rank": rank + 1}

@app.get("/leaderboard")
def get_leaderboard(top: int = 10):
    results = r.zrevrange("leaderboard", 0, top - 1, withscores=True)
    return {
        "leaderboard": [
            {"rank": i + 1, "player": name.decode(), "score": score}
            for i, (name, score) in enumerate(results)
        ]
    }

@app.post("/leaderboard/{player}/increment")
def increment_score(player: str, points: float):
    new_score = r.zincrby("leaderboard", points, player)
    return {"player": player, "new_score": new_score}
```

`r.zadd()` adds a player with a score. `r.zrevrange()` returns the top players sorted from highest to lowest. `r.zincrby()` atomically increments a player's score.

---

### Hashes

Hashes are maps of field-value pairs, ideal for representing objects. They are more memory-efficient than storing each field as a separate key.

```bash
HSET user:1 name "Bidur" age 25 email "bidur@example.com"
HGET user:1 name                       # Get single field → "Bidur"
HGETALL user:1                         # Get all fields and values
HMGET user:1 name email                # Get multiple fields

HSET user:1 age 26                     # Update a field
HSETNX user:1 country "Nepal"          # Set only if field does not exist

HDEL user:1 email                      # Delete a field
HEXISTS user:1 name                    # Check if field exists → 1
HLEN user:1                            # Number of fields
HKEYS user:1                           # Get all field names
HVALS user:1                           # Get all values

HINCRBY user:1 age 1                   # Increment integer field by 1
HINCRBYFLOAT user:1 balance 9.99       # Increment float field

HSCAN user:1 0 MATCH "na*" COUNT 10    # Scan fields matching pattern
```

`HSET` sets one or more field-value pairs in a single command. `HGETALL` returns every field and value as a flat list (field1, value1, field2, value2, ...). `HSETNX` is atomic and only sets the field if it does not already exist. `HINCRBY` treats the field value as an integer and increments it atomically. The `user:1` naming convention uses colons as namespace separators — this is a widely adopted Redis convention for organizing keys.

### FastAPI Example: Hashes (User Profile)

```python
class UserProfile(BaseModel):
    name: str
    email: str
    age: int

@app.post("/users/{user_id}")
def create_user(user_id: int, profile: UserProfile):
    r.hset(f"user:{user_id}", mapping=profile.model_dump())
    return {"user_id": user_id, "created": True}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    data = r.hgetall(f"user:{user_id}")
    if not data:
        return {"error": "user not found"}
    return {k.decode(): v.decode() for k, v in data.items()}

@app.patch("/users/{user_id}")
def update_user(user_id: int, field: str, value: str):
    r.hset(f"user:{user_id}", field, value)
    return {"user_id": user_id, "updated": {field: value}}
```

`r.hset(mapping=...)` sets multiple fields at once. `r.hgetall()` returns all fields as a dictionary. Each field is updated independently without rewriting the whole object.

---

### Bitmaps

Bitmaps are not a separate data type but a set of bit-oriented operations on strings. They are extremely memory-efficient for tracking boolean states across large populations.

```bash
SETBIT logins:2025-01-15 1001 1        # Mark user 1001 as logged in
GETBIT logins:2025-01-15 1001          # Check if logged in → 1
BITCOUNT logins:2025-01-15             # Count all set bits (active users)

BITOP AND result day1 day2             # Users active on BOTH days
BITOP OR result day1 day2              # Users active on EITHER day
BITOP XOR result day1 day2             # Users active on exactly one day

BITPOS logins:2025-01-15 1             # Position of first set bit
BITPOS logins:2025-01-15 0             # Position of first unset bit
```

`SETBIT` sets the bit at the given offset to 0 or 1. The offset is a user ID or any integer identifier. `BITCOUNT` counts the number of bits set to 1. `BITOP` performs bitwise operations across multiple keys, useful for computing daily/weekly active users. A bitmap tracking 100 million users consumes only ~12.5 MB of memory.

### FastAPI Example: Bitmaps (Daily Active Users)

```python
from datetime import date

@app.post("/track/login/{user_id}")
def track_login(user_id: int):
    today = date.today().isoformat()
    r.setbit(f"logins:{today}", user_id, 1)
    return {"user_id": user_id, "tracked": today}

@app.get("/stats/daily-active")
def daily_active_users():
    today = date.today().isoformat()
    count = r.bitcount(f"logins:{today}")
    return {"date": today, "active_users": count}

@app.get("/track/check/{user_id}")
def check_login(user_id: int):
    today = date.today().isoformat()
    logged_in = r.getbit(f"logins:{today}", user_id)
    return {"user_id": user_id, "logged_in_today": bool(logged_in)}
```

`r.setbit()` marks a user as active using their ID as the bit offset. `r.bitcount()` counts how many users are active. A single key uses minimal memory even with millions of users.

---

### HyperLogLog

HyperLogLog is a probabilistic data structure used for counting unique elements (cardinality estimation) with a standard error of 0.81%. It uses a fixed ~12 KB of memory regardless of the number of elements.

```bash
PFADD visitors "user1" "user2" "user3" # Add elements
PFADD visitors "user1"                 # Duplicates are ignored
PFCOUNT visitors                       # Approximate count → 3

PFADD visitors:page1 "u1" "u2" "u3"
PFADD visitors:page2 "u2" "u3" "u4"
PFMERGE visitors:total visitors:page1 visitors:page2
PFCOUNT visitors:total                 # Approximate union count → 4
```

`PFADD` adds elements to the HyperLogLog; it returns 1 if the internal representation changed, 0 otherwise. `PFCOUNT` returns the approximate cardinality. `PFMERGE` merges multiple HyperLogLogs into one. HyperLogLogs are ideal for counting unique visitors, unique searches, or unique events where exact counts are not critical and memory efficiency matters.

### FastAPI Example: HyperLogLog (Unique Visitor Counter)

```python
@app.post("/visit/{page}")
def record_visit(page: str, visitor_id: str):
    r.pfadd(f"visitors:{page}", visitor_id)
    return {"page": page, "visitor": visitor_id}

@app.get("/visit/{page}/count")
def unique_visitor_count(page: str):
    count = r.pfcount(f"visitors:{page}")
    return {"page": page, "unique_visitors_approx": count}

@app.get("/visit/total")
def total_unique_visitors(pages: str):
    page_list = pages.split(",")
    keys = [f"visitors:{p}" for p in page_list]
    r.pfmerge("visitors:merged", *keys)
    count = r.pfcount("visitors:merged")
    return {"pages": page_list, "total_unique_approx": count}
```

`r.pfadd()` adds a visitor to the HyperLogLog. `r.pfcount()` returns the approximate count of unique visitors. `r.pfmerge()` combines counts across pages for a total.

---

### Streams

Streams are an append-only log data structure, similar to Kafka topics. They support consumer groups for distributed message processing.

```bash
XADD events * sensor "temp" value "22.5"   # Add entry (auto ID)
XADD events 1609459200000-0 msg "hello"    # Add with explicit ID
XLEN events                                 # Number of entries

XRANGE events - +                           # Read all entries
XRANGE events - + COUNT 10                  # Read first 10 entries
XREVRANGE events + -                        # Read in reverse order
XREAD COUNT 5 STREAMS events 0              # Read from beginning
XREAD BLOCK 5000 STREAMS events $           # Block for new entries (5s timeout)

# Consumer groups
XGROUP CREATE events mygroup 0              # Create consumer group
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS events >  # Read new messages
XACK events mygroup 1609459200000-0         # Acknowledge processed message

XINFO STREAM events                         # Stream info
XINFO GROUPS events                         # Consumer group info
XTRIM events MAXLEN 1000                    # Trim to last 1000 entries
XDEL events 1609459200000-0                 # Delete specific entry
```

`XADD` appends a new entry; `*` auto-generates a time-based ID. Each entry is a set of field-value pairs. `XRANGE - +` reads from the earliest (`-`) to the latest (`+`) entry. `XREAD BLOCK` waits for new messages, making it suitable for real-time event processing. Consumer groups allow multiple consumers to cooperatively process messages without duplication. `XACK` acknowledges that a consumer has processed a message. `XTRIM MAXLEN` keeps the stream capped at a maximum number of entries.

### FastAPI Example: Streams (Event Log)

```python
@app.post("/events")
def add_event(event_type: str, data: str):
    entry_id = r.xadd("events", {"type": event_type, "data": data})
    return {"event_id": entry_id.decode(), "type": event_type}

@app.get("/events")
def get_events(count: int = 10):
    entries = r.xrevrange("events", "+", "-", count=count)
    return {
        "events": [
            {"id": eid.decode(), "fields": {k.decode(): v.decode() for k, v in fields.items()}}
            for eid, fields in entries
        ]
    }

@app.get("/events/length")
def event_count():
    return {"total_events": r.xlen("events")}
```

`r.xadd()` appends an event and returns its auto-generated ID. `r.xrevrange()` reads the most recent events first. `r.xlen()` returns the total number of entries.

---

### Geospatial

Redis supports geospatial indexing using sorted sets internally. You can store coordinates and query by radius or bounding box.

```bash
GEOADD locations 85.3240 27.7172 "Kathmandu"
GEOADD locations 77.2090 28.6139 "Delhi" 88.3639 22.5726 "Kolkata"

GEOPOS locations "Kathmandu"           # Get coordinates
GEODIST locations "Kathmandu" "Delhi" km   # Distance → ~880 km
GEOSEARCH locations FROMMEMBER "Kathmandu" BYRADIUS 1000 km ASC
GEOSEARCH locations FROMLONLAT 85.0 27.5 BYRADIUS 100 km WITHCOORD WITHDIST
```

`GEOADD` stores longitude, latitude, and member name. `GEODIST` calculates the distance between two members in the specified unit (m, km, mi, ft). `GEOSEARCH` finds members within a radius or bounding box from a given point or member.

### FastAPI Example: Geospatial (Nearby Locations)

```python
@app.post("/locations")
def add_location(name: str, lon: float, lat: float):
    r.geoadd("locations", (lon, lat, name))
    return {"added": name, "lon": lon, "lat": lat}

@app.get("/locations/nearby")
def nearby(lon: float, lat: float, radius_km: float = 10):
    results = r.geosearch(
        "locations",
        longitude=lon,
        latitude=lat,
        radius=radius_km,
        unit="km",
        withcoord=True,
        withdist=True,
        sort="ASC",
    )
    return {
        "nearby": [
            {"name": name.decode(), "distance_km": dist, "lon": coord[0], "lat": coord[1]}
            for name, dist, coord in results
        ]
    }

@app.get("/locations/distance")
def distance(place1: str, place2: str):
    dist = r.geodist("locations", place1, place2, unit="km")
    return {"from": place1, "to": place2, "distance_km": dist}
```

`r.geoadd()` stores a location with longitude, latitude, and name. `r.geosearch()` finds nearby locations sorted by distance. `r.geodist()` computes the distance between two stored locations.

---

## Key Management

Keys are the foundation of Redis. These commands work across all data types.

### Basic Key Operations

```bash
SET greeting "hello"
EXISTS greeting                        # Check if key exists → 1
TYPE greeting                          # Get data type → string
RENAME greeting welcome                # Rename key
RENAMENX greeting welcome              # Rename only if "welcome" doesn't exist

DEL welcome                            # Delete key (blocking)
UNLINK welcome                         # Delete key (non-blocking, async)

OBJECT ENCODING greeting               # Internal encoding (e.g., embstr)
OBJECT REFCOUNT greeting               # Reference count
OBJECT IDLETIME greeting               # Seconds since last access
DEBUG OBJECT greeting                  # Detailed internal info
```

`DEL` deletes keys synchronously; for large keys (millions of elements), it can block the server. `UNLINK` deletes keys asynchronously in a background thread, which is preferred for large keys. `OBJECT ENCODING` shows the internal data structure Redis uses (e.g., `embstr`, `ziplist`, `hashtable`).

### Key Expiration

```bash
SET session "data"
EXPIRE session 3600                    # Expire in 3600 seconds
PEXPIRE session 5000                   # Expire in 5000 milliseconds
EXPIREAT session 1735689600            # Expire at Unix timestamp
PEXPIREAT session 1735689600000        # Expire at Unix timestamp (ms)

TTL session                            # Remaining time in seconds
PTTL session                           # Remaining time in milliseconds
PERSIST session                        # Remove expiration (make permanent)
```

`EXPIRE` sets a timeout on a key. Once the timeout expires, the key is automatically deleted. `TTL` returns `-1` if the key has no expiration, `-2` if the key does not exist, or the remaining seconds. `PERSIST` removes the expiration, making the key permanent again.

### Searching Keys

```bash
KEYS user:*                            # Find keys matching pattern (SLOW)
KEYS *                                 # List all keys (NEVER in production)
SCAN 0 MATCH user:* COUNT 100         # Incrementally iterate keys (SAFE)
```

`KEYS` scans the entire keyspace and blocks the server while running. In production, **never use `KEYS`** — use `SCAN` instead. `SCAN` returns a cursor and a batch of keys; call it repeatedly with the returned cursor until it returns 0. `COUNT` is a hint for how many keys to return per call (not a guarantee).

### SCAN Iteration Pattern

```bash
SCAN 0 MATCH user:* COUNT 100         # First call → returns cursor + keys
SCAN <cursor> MATCH user:* COUNT 100   # Next call with returned cursor
# Continue until cursor returns 0
```

Each `SCAN` call returns two things: a new cursor and a list of matching keys. Start with cursor `0` and keep calling with the returned cursor until it returns `0`. This ensures full iteration without blocking the server.

### SORT

```bash
RPUSH scores 5 3 8 1 4
SORT scores                            # Sort numerically → 1 3 4 5 8
SORT scores DESC                       # Sort descending → 8 5 4 3 1
SORT scores LIMIT 0 3                  # Sort and return first 3
SORT scores ALPHA                      # Sort alphabetically
SORT scores STORE sorted_scores        # Sort and store result
```

`SORT` sorts elements in a list, set, or sorted set. `LIMIT offset count` paginates the result. `ALPHA` treats values as strings for lexicographic sorting. `STORE` saves the sorted result to a new key as a list.

### FastAPI Example: Key Management (Session Store)

```python
import json
import uuid

@app.post("/sessions")
def create_session(username: str):
    session_id = str(uuid.uuid4())
    session_data = json.dumps({"username": username, "created": str(date.today())})
    r.setex(f"session:{session_id}", 3600, session_data)  # expires in 1 hour
    return {"session_id": session_id, "expires_in": 3600}

@app.get("/sessions/{session_id}")
def get_session(session_id: str):
    data = r.get(f"session:{session_id}")
    if not data:
        return {"error": "session expired or not found"}
    ttl = r.ttl(f"session:{session_id}")
    return {"session": json.loads(data), "ttl_seconds": ttl}

@app.delete("/sessions/{session_id}")
def delete_session(session_id: str):
    deleted = r.delete(f"session:{session_id}")
    return {"deleted": bool(deleted)}
```

`r.setex()` creates a key with an automatic expiration. `r.ttl()` checks remaining time before the key is deleted. `r.delete()` removes the key immediately.

---

## Pub/Sub Messaging

Redis Pub/Sub provides a publish/subscribe messaging system. Publishers send messages to channels, and subscribers receive messages from channels they are subscribed to. Messages are fire-and-forget — if no subscriber is listening, the message is lost.

### Subscribing

```bash
docker exec -it redis redis-cli SUBSCRIBE news                  # Subscribe to "news" channel
docker exec -it redis redis-cli SUBSCRIBE news sports weather   # Subscribe to multiple
docker exec -it redis redis-cli PSUBSCRIBE "news:*"             # Subscribe to pattern
```

`SUBSCRIBE` blocks the connection and waits for messages. Once subscribed, the connection can only run subscribe-related commands. `PSUBSCRIBE` uses glob-style patterns to match multiple channels.

### Publishing

```bash
docker exec -it redis redis-cli PUBLISH news "Breaking news!"
docker exec -it redis redis-cli PUBLISH news:sports "Goal scored!"
```

`PUBLISH` sends a message to all subscribers of the given channel and returns the number of clients that received the message.

### Unsubscribing

```bash
UNSUBSCRIBE news                       # Unsubscribe from specific channel
PUNSUBSCRIBE news:*                    # Unsubscribe from pattern
```

### Pub/Sub Considerations

Pub/Sub in Redis has important limitations:

- **No persistence**: Messages are not stored. If a subscriber is offline, it misses messages.
- **No acknowledgment**: There is no way to confirm a subscriber received a message.
- **No replay**: You cannot replay past messages.
- **At-most-once delivery**: Messages may be lost if a subscriber disconnects.

For durable messaging, use Redis Streams instead of Pub/Sub.

### FastAPI Example: Pub/Sub (Notifications)

```python
import asyncio
import redis.asyncio as aioredis
from fastapi.responses import StreamingResponse

# Async Redis connection for Pub/Sub
async_redis = aioredis.from_url(os.getenv("REDIS_URL", "redis://localhost:6379/0"))

@app.post("/notify/{channel}")
def publish_message(channel: str, message: str):
    listeners = r.publish(channel, message)
    return {"channel": channel, "message": message, "listeners": listeners}

@app.get("/subscribe/{channel}")
async def subscribe(channel: str):
    async def event_stream():
        pubsub = async_redis.pubsub()
        await pubsub.subscribe(channel)
        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    yield f"data: {message['data'].decode()}\n\n"
        finally:
            await pubsub.unsubscribe(channel)

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

The `/notify/{channel}` endpoint publishes a message. The `/subscribe/{channel}` endpoint uses Server-Sent Events (SSE) to stream messages to the client in real time. `r.publish()` sends to all subscribers and returns how many received it. Note: add `redis[hiredis]` to requirements.txt for the async client.

---

## Transactions

Redis transactions group multiple commands into a single atomic unit. All commands in a transaction execute sequentially without interruption from other clients.

### MULTI / EXEC

```bash
MULTI                                  # Start transaction
SET balance 100
DECRBY balance 30
INCRBY balance 10
EXEC                                   # Execute all commands atomically
```

`MULTI` starts a transaction. After `MULTI`, all subsequent commands are queued (not executed). `EXEC` executes all queued commands atomically. If you want to cancel, use `DISCARD` instead of `EXEC`.

### WATCH (Optimistic Locking)

```bash
WATCH balance                          # Watch key for changes
MULTI
DECRBY balance 50
EXEC                                   # Fails (returns nil) if "balance" changed
```

`WATCH` monitors one or more keys. If any watched key is modified by another client before `EXEC`, the transaction is aborted and `EXEC` returns `nil`. This implements optimistic locking — you check if conditions still hold before committing. `UNWATCH` cancels all watches.

### Transaction Rules

- Commands between `MULTI` and `EXEC` are queued, not executed immediately.
- If any command has a syntax error, the entire transaction is discarded.
- If a command fails at runtime (e.g., wrong type), other commands in the transaction still execute (no rollback).
- Redis transactions do **not** support rollback. If a command fails, it is the programmer's error.

### FastAPI Example: Transactions (Fund Transfer)

```python
@app.post("/transfer")
def transfer(from_user: str, to_user: str, amount: float):
    from_key = f"balance:{from_user}"
    to_key = f"balance:{to_user}"

    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(from_key)
                balance = float(pipe.get(from_key) or 0)
                if balance < amount:
                    return {"error": "insufficient funds"}

                pipe.multi()
                pipe.decrby(from_key, int(amount))
                pipe.incrby(to_key, int(amount))
                pipe.execute()
                return {"transferred": amount, "from": from_user, "to": to_user}
            except redis.WatchError:
                continue  # Retry if another client modified the balance

@app.post("/balance/{user}")
def set_balance(user: str, amount: float):
    r.set(f"balance:{user}", int(amount))
    return {"user": user, "balance": amount}
```

`r.pipeline()` creates a pipeline. `pipe.watch()` watches a key for changes. `pipe.multi()` starts the transaction. If another client modifies the balance between `watch` and `execute`, a `WatchError` is raised and the loop retries. This is the Redis pattern for optimistic locking.

---

## Lua Scripting

Redis supports server-side Lua scripting with `EVAL`. Scripts execute atomically on the server, reducing round trips and ensuring no other command runs during the script.

### Basic Scripting

```bash
EVAL "return 'Hello, Redis!'" 0
# → "Hello, Redis!"

EVAL "return redis.call('GET', KEYS[1])" 1 mykey
# → value of "mykey"

EVAL "redis.call('SET', KEYS[1], ARGV[1]); return redis.call('GET', KEYS[1])" 1 greeting "hello"
# → "hello"
```

`EVAL` takes the script, the number of key arguments, the keys, and additional arguments. `KEYS` is a 1-indexed Lua table of key names. `ARGV` is a 1-indexed table of additional arguments. `redis.call()` runs a Redis command and raises an error on failure. `redis.pcall()` is the same but returns errors as Lua tables instead of raising them.

### Script Caching

```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → returns SHA1 hash: "a42059b3..."

EVALSHA "a42059b3..." 1 mykey          # Execute cached script by SHA1

SCRIPT EXISTS "a42059b3..."            # Check if script is cached → 1
SCRIPT FLUSH                           # Clear script cache
```

`SCRIPT LOAD` sends the script to Redis and returns a SHA1 hash. `EVALSHA` executes a cached script by its SHA1 hash, avoiding the overhead of sending the full script each time. This is the recommended approach in production.

### Example: Rate Limiter

```bash
EVAL "
  local current = redis.call('INCR', KEYS[1])
  if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
  end
  return current
" 1 ratelimit:user:42 60
```

This script atomically increments a counter and sets an expiration on the first increment. It returns the current count, which the application can compare against a limit. The key expires after 60 seconds, resetting the counter.

### FastAPI Example: Lua Scripting (Rate Limiter)

```python
from fastapi import HTTPException

RATE_LIMIT_SCRIPT = """
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return current
"""

# Register the script once at startup
rate_limit_sha = r.script_load(RATE_LIMIT_SCRIPT)

@app.get("/api/data")
def get_data(user_id: str):
    key = f"ratelimit:{user_id}"
    window_seconds = 60
    max_requests = 10

    current = r.evalsha(rate_limit_sha, 1, key, window_seconds)
    if current > max_requests:
        ttl = r.ttl(key)
        raise HTTPException(status_code=429, detail=f"Rate limited. Retry in {ttl}s")

    return {"data": "here is your data", "requests_used": current, "limit": max_requests}
```

`r.script_load()` caches the Lua script and returns its SHA1 hash. `r.evalsha()` executes the cached script efficiently. The script atomically increments a counter and sets its TTL on the first request, ensuring the rate limit window is precise.

---

## Persistence

Redis offers two persistence mechanisms to save in-memory data to disk. You can use either one, both together, or neither.

### RDB (Redis Database Snapshots)

RDB creates point-in-time snapshots of the entire dataset at specified intervals.

```conf
# redis.conf
save 900 1                             # Snapshot if 1+ key changed in 900 seconds
save 300 10                            # Snapshot if 10+ keys changed in 300 seconds
save 60 10000                          # Snapshot if 10000+ keys changed in 60 seconds
dbfilename dump.rdb                    # Snapshot filename
dir /var/lib/redis                     # Directory for RDB file
```

```bash
BGSAVE                                 # Trigger background snapshot manually
SAVE                                   # Trigger foreground snapshot (blocks server)
LASTSAVE                               # Unix timestamp of last successful snapshot
```

`BGSAVE` forks the Redis process to write the snapshot in the background without blocking the main server. `SAVE` blocks the server during the snapshot — only use in maintenance windows. RDB files are compact and ideal for backups, but you may lose data between snapshots.

### AOF (Append Only File)

AOF logs every write operation to a file. On restart, Redis replays the AOF to reconstruct the dataset.

```conf
# redis.conf
appendonly yes                         # Enable AOF
appendfilename "appendonly.aof"        # AOF filename
appendfsync everysec                   # Sync to disk every second (recommended)
# appendfsync always                   # Sync after every write (safest, slowest)
# appendfsync no                       # Let OS decide when to sync (fastest)
```

```bash
BGREWRITEAOF                           # Rewrite AOF in background (compaction)
```

`appendfsync everysec` provides a good balance of performance and durability, with at most 1 second of data loss. `always` is the safest but slowest because it syncs after every single write. `BGREWRITEAOF` compacts the AOF file by creating a minimal set of commands that reconstruct the current dataset.

### RDB vs AOF Comparison

| Feature        | RDB                        | AOF                        |
| -------------- | -------------------------- | -------------------------- |
| Data Loss Risk | Data between snapshots     | At most 1 second (default) |
| File Size      | Compact                    | Larger (full command log)  |
| Restart Speed  | Faster (load binary)       | Slower (replay commands)   |
| Write Impact   | Fork + write (periodic)    | Continuous writes          |
| Best For       | Backups, disaster recovery | Durability, minimal loss   |

### Recommended: Both RDB + AOF

```conf
appendonly yes
save 900 1
save 300 10
save 60 10000
```

Using both gives you the best of both worlds: RDB for fast restarts and backups, AOF for minimal data loss. If both are present at startup, Redis loads the AOF because it is typically more complete.

### Docker: Persistence with Custom Config

```bash
# Create redis.conf on the host
cat > redis.conf <<EOF
appendonly yes
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /data
EOF

# Run with persistence
docker run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  -v ./redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7 redis-server /usr/local/etc/redis/redis.conf
```

The named volume `redis-data` keeps the RDB and AOF files across container restarts.

### FastAPI Example: Persistence Info

```python
@app.get("/redis/persistence")
def persistence_info():
    info = r.info("persistence")
    return {
        "rdb_last_save_time": info.get("rdb_last_save_time"),
        "rdb_changes_since_last_save": info.get("rdb_changes_since_last_save"),
        "aof_enabled": info.get("aof_enabled"),
        "aof_current_size": info.get("aof_current_size"),
        "aof_last_rewrite_status": info.get("aof_last_rewrite_status"),
    }

@app.post("/redis/bgsave")
def trigger_snapshot():
    r.bgsave()
    return {"message": "background save triggered"}
```

`r.info("persistence")` returns persistence-related stats. `r.bgsave()` triggers a background RDB snapshot. This is useful for admin dashboards or health monitoring.

---

## Replication

Redis replication creates copies (replicas) of a master Redis instance. Replicas maintain an exact copy of the master's data and can serve read requests, distributing the read load.

### Docker Compose: Master-Replica Setup

```yaml
# docker-compose-replication.yml
services:
  redis-master:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - master-data:/data
    command: redis-server --appendonly yes

  redis-replica:
    image: redis:7
    ports:
      - "6380:6379"
    volumes:
      - replica-data:/data
    command: redis-server --replicaof redis-master 6379 --replica-read-only yes
    depends_on:
      - redis-master

volumes:
  master-data:
  replica-data:
```

`--replicaof redis-master 6379` tells the replica to follow the master using Docker's internal DNS. `--replica-read-only yes` prevents writes to the replica (default and recommended).

### How Replication Works

1. Replica connects to the master and requests a full synchronization.
2. Master starts a `BGSAVE` to create an RDB snapshot.
3. Master sends the RDB file to the replica.
4. Master sends all new write commands that occurred during the snapshot to the replica.
5. From this point, master sends a continuous stream of write commands to the replica in real time.

Replication is asynchronous by default: the master does not wait for replicas to acknowledge writes. Use `WAIT` to optionally block until replicas confirm.

```bash
WAIT 1 5000                            # Wait for 1 replica to ack, timeout 5s
```

### Verifying Replication

```bash
# Check master status
docker exec -it redis-master redis-cli INFO replication
# → role:master, connected_slaves:1

# Check replica status
docker exec -it redis-replica redis-cli INFO replication
# → role:slave, master_host:redis-master

# Write to master, read from replica
docker exec -it redis-master redis-cli SET test "hello"
docker exec -it redis-replica redis-cli GET test
# → "hello"
```

### Promoting a Replica

```bash
docker exec -it redis-replica redis-cli REPLICAOF NO ONE
# Now redis-replica is an independent master
```

### FastAPI Example: Replication (Read/Write Split)

```python
import os
import redis

# Write to master, read from replica
writer = redis.from_url(os.getenv("REDIS_MASTER_URL", "redis://localhost:6379/0"))
reader = redis.from_url(os.getenv("REDIS_REPLICA_URL", "redis://localhost:6380/0"))

@app.post("/cache/{key}")
def write_cache(key: str, value: str):
    writer.set(f"cache:{key}", value)
    return {"key": key, "written_to": "master"}

@app.get("/cache/{key}")
def read_cache(key: str):
    value = reader.get(f"cache:{key}")
    if not value:
        return {"error": "not found"}
    return {"key": key, "value": value.decode(), "read_from": "replica"}

@app.get("/replication/status")
def replication_status():
    master_info = writer.info("replication")
    replica_info = reader.info("replication")
    return {
        "master_role": master_info["role"],
        "connected_slaves": master_info.get("connected_slaves"),
        "replica_role": replica_info["role"],
        "master_link_status": replica_info.get("master_link_status"),
    }
```

Writes go to the master. Reads go to the replica, offloading read traffic from the master. The `replication/status` endpoint monitors the health of the replication link.

---

## Redis Sentinel

Sentinel provides high availability for Redis. It monitors master and replica instances, detects failures, and performs automatic failover.

### Docker Compose: Sentinel Setup

```yaml
# docker-compose-sentinel.yml
services:
  redis-master:
    image: redis:7
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes

  redis-replica-1:
    image: redis:7
    ports:
      - "6380:6379"
    command: redis-server --replicaof redis-master 6379
    depends_on:
      - redis-master

  redis-replica-2:
    image: redis:7
    ports:
      - "6381:6379"
    command: redis-server --replicaof redis-master 6379
    depends_on:
      - redis-master

  sentinel-1:
    image: redis:7
    ports:
      - "26379:26379"
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    command: redis-sentinel /etc/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2

  sentinel-2:
    image: redis:7
    ports:
      - "26380:26379"
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    command: redis-sentinel /etc/sentinel.conf
    depends_on:
      - redis-master

  sentinel-3:
    image: redis:7
    ports:
      - "26381:26379"
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    command: redis-sentinel /etc/sentinel.conf
    depends_on:
      - redis-master

volumes:
  master-data:
```

### Sentinel Config

```conf
# sentinel.conf
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

`sentinel monitor` defines the master to monitor; the last number (`2`) is the quorum — the minimum number of Sentinels that must agree a master is down before failover. `down-after-milliseconds` is how long a master must be unreachable before it is considered down. `failover-timeout` sets the maximum time for the failover process.

### Sentinel Commands

```bash
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL masters
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL replicas mymaster
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
docker exec -it sentinel-1 redis-cli -p 26379 SENTINEL failover mymaster   # Force failover
```

In production, run at least 3 Sentinel instances to form a reliable quorum. Clients connect to Sentinel first to discover the current master, then connect to the master directly.

### FastAPI Example: Sentinel (Auto-Failover Connection)

```python
from redis.sentinel import Sentinel

sentinels = [("localhost", 26379), ("localhost", 26380), ("localhost", 26381)]
sentinel = Sentinel(sentinels, socket_timeout=0.5)

@app.get("/sentinel/master")
def get_master():
    master_addr = sentinel.discover_master("mymaster")
    return {"master_host": master_addr[0], "master_port": master_addr[1]}

@app.get("/sentinel/replicas")
def get_replicas():
    replicas = sentinel.discover_slaves("mymaster")
    return {"replicas": [{"host": h, "port": p} for h, p in replicas]}

@app.post("/sentinel/write/{key}")
def sentinel_write(key: str, value: str):
    master = sentinel.master_for("mymaster", socket_timeout=0.5)
    master.set(key, value)
    return {"key": key, "value": value, "written_to": "current master"}

@app.get("/sentinel/read/{key}")
def sentinel_read(key: str):
    slave = sentinel.slave_for("mymaster", socket_timeout=0.5)
    value = slave.get(key)
    return {"key": key, "value": value.decode() if value else None, "read_from": "replica"}
```

`Sentinel()` connects to the Sentinel cluster. `sentinel.master_for()` returns a Redis client pointing to the current master — if the master fails over, the client automatically reconnects to the new master. `sentinel.slave_for()` returns a client pointing to a replica for reads.

---

## Redis Cluster

Redis Cluster provides automatic sharding across multiple Redis nodes. Data is distributed using hash slots — the key space is divided into 16,384 slots, and each node is responsible for a subset of slots.

### How Cluster Works

- The keyspace is divided into **16,384 hash slots**.
- Each master node owns a subset of slots.
- The hash slot for a key is computed as `CRC16(key) mod 16384`.
- Each master can have one or more replicas for failover.
- Clients are redirected to the correct node if they send a command to the wrong one.

### Docker Compose: Redis Cluster

```yaml
# docker-compose-cluster.yml
services:
  redis-node-1:
    image: redis:7
    ports:
      - "7000:7000"
      - "17000:17000"
    volumes:
      - ./redis-cluster-7000.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-2:
    image: redis:7
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - ./redis-cluster-7001.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-3:
    image: redis:7
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - ./redis-cluster-7002.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-4:
    image: redis:7
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - ./redis-cluster-7003.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-5:
    image: redis:7
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - ./redis-cluster-7004.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-6:
    image: redis:7
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - ./redis-cluster-7005.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
```

### Cluster Node Config

```conf
# redis-cluster-7000.conf (repeat for each port)
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes
```

### Creating the Cluster

```bash
docker compose -f docker-compose-cluster.yml up -d

# Create the cluster (run from any node)
docker exec -it redis-node-1 redis-cli --cluster create \
  redis-node-1:7000 redis-node-2:7001 redis-node-3:7002 \
  redis-node-4:7003 redis-node-5:7004 redis-node-6:7005 \
  --cluster-replicas 1
```

`--cluster-replicas 1` assigns one replica per master. With 6 nodes and 1 replica each, you get 3 masters and 3 replicas.

### Cluster Commands

```bash
docker exec -it redis-node-1 redis-cli -c -p 7000    # Connect in cluster mode
CLUSTER INFO                           # Cluster status
CLUSTER NODES                          # List all nodes and their slots
CLUSTER SLOTS                          # Slot-to-node mapping
CLUSTER KEYSLOT mykey                  # Which slot does "mykey" belong to?

docker exec -it redis-node-1 redis-cli --cluster check redis-node-1:7000     # Health check
docker exec -it redis-node-1 redis-cli --cluster reshard redis-node-1:7000   # Redistribute slots
```

`-c` enables cluster mode in `redis-cli`, which automatically follows `MOVED` and `ASK` redirections. `reshard` moves hash slots between nodes for rebalancing.

### Hash Tags

```bash
SET {user:1}.name "Bidur"
SET {user:1}.email "bidur@example.com"
```

Hash tags (`{...}`) ensure that keys with the same tag hash to the same slot, so they can be accessed together in multi-key operations. Only the substring inside `{}` is used for hashing. This is required for multi-key commands like `MGET` in a cluster.

### FastAPI Example: Cluster (Sharded Access)

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    startup_nodes=[
        {"host": "localhost", "port": 7000},
        {"host": "localhost", "port": 7001},
        {"host": "localhost", "port": 7002},
    ],
    decode_responses=True,
)

@app.post("/cluster/set/{key}")
def cluster_set(key: str, value: str):
    rc.set(key, value)
    slot = rc.keyslot(key)
    return {"key": key, "value": value, "hash_slot": slot}

@app.get("/cluster/get/{key}")
def cluster_get(key: str):
    value = rc.get(key)
    slot = rc.keyslot(key)
    return {"key": key, "value": value, "hash_slot": slot}

@app.get("/cluster/info")
def cluster_info():
    info = rc.cluster_info()
    return {
        "cluster_state": info.get("cluster_state"),
        "cluster_slots_assigned": info.get("cluster_slots_assigned"),
        "cluster_known_nodes": info.get("cluster_known_nodes"),
    }
```

`RedisCluster()` automatically discovers all nodes from the startup nodes. `rc.keyslot()` shows which hash slot a key maps to. The client handles `MOVED` redirections transparently — you interact with the cluster as if it were a single server.

---

## Security

### Authentication

```conf
# redis.conf
requirepass yourstrongpassword         # Set password
# For Redis 6+ ACL-based auth:
user default on >yourpassword ~* +@all
```

```bash
docker exec -it redis redis-cli
AUTH yourpassword                      # Authenticate after connecting
docker exec -it redis redis-cli -a yourpassword   # Authenticate on connect
```

`requirepass` sets a single server-wide password. Redis 6+ introduced ACLs (Access Control Lists) for more granular permissions with username-based authentication.

### Docker: Running Redis with Password

```bash
docker run -d --name redis -p 6379:6379 \
  redis:7 redis-server --requirepass yourstrongpassword
```

### ACL (Access Control Lists)

```bash
ACL SETUSER appuser on >apppassword ~app:* +get +set +del
ACL SETUSER readonly on >readpass ~* +get +mget +keys +scan -@write
ACL LIST                               # List all users and rules
ACL GETUSER appuser                    # Get specific user details
ACL DELUSER appuser                    # Delete a user
ACL WHOAMI                             # Current authenticated user
```

`ACL SETUSER` creates or modifies a user. `on` enables the user. `>password` sets the password. `~app:*` restricts accessible keys to those matching the pattern. `+get +set` allows specific commands. `-@write` denies all commands in the write category.

### Network Security

```conf
# redis.conf
bind 127.0.0.1                         # Listen only on localhost
protected-mode yes                     # Reject external connections without auth
rename-command FLUSHALL ""             # Disable dangerous commands
rename-command CONFIG ""               # Disable CONFIG command
rename-command DEBUG ""                # Disable DEBUG command
```

`bind 127.0.0.1` prevents Redis from accepting connections from external networks. `protected-mode yes` blocks connections from non-loopback interfaces if no password is set. `rename-command ""` effectively disables a command by renaming it to an empty string.

### TLS/SSL Encryption

```conf
# redis.conf
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```

```bash
redis-cli --tls --cert /path/to/client.crt --key /path/to/client.key --cacert /path/to/ca.crt -p 6380
```

TLS encrypts all communication between clients and the Redis server. This is essential when Redis is exposed over a network.

### FastAPI Example: Security (Authenticated Connection)

```python
import os
import redis

# Connect with password
r = redis.Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD", "yourstrongpassword"),
    decode_responses=True,
)

# Or with ACL username + password
r_acl = redis.Redis(
    host="redis",
    port=6379,
    username="appuser",
    password="apppassword",
    decode_responses=True,
)

@app.get("/secure/ping")
def secure_ping():
    return {"ping": r.ping(), "authenticated": True}
```

The `password` parameter authenticates using `requirepass`. The `username` + `password` parameters authenticate using ACL. Use `decode_responses=True` to get strings instead of bytes.

---

## Performance & Optimization

### Memory Optimization

```bash
INFO memory                            # Check memory usage
MEMORY USAGE mykey                     # Memory used by a specific key
MEMORY DOCTOR                          # Memory health diagnostics
OBJECT ENCODING mykey                  # Check internal encoding
```

Redis uses optimized internal encodings for small data structures:

| Type       | Small Encoding | Large Encoding | Threshold               |
| ---------- | -------------- | -------------- | ----------------------- |
| String     | embstr / int   | raw            | 44 bytes                |
| List       | listpack       | quicklist      | 128 elements / 64 bytes |
| Set        | listpack       | hashtable      | 128 elements / 64 bytes |
| Sorted Set | listpack       | skiplist       | 128 elements / 64 bytes |
| Hash       | listpack       | hashtable      | 128 elements / 64 bytes |

Keep entries small to benefit from compact encodings. The thresholds are configurable via `*-max-listpack-entries` and `*-max-listpack-value` settings.

### Pipelining

Pipelining sends multiple commands to Redis without waiting for individual replies, reducing network round trips.

```bash
# Without pipelining: 3 round trips
SET a 1     # → OK
SET b 2     # → OK
SET c 3     # → OK

# With pipelining (redis-cli):
redis-cli --pipe <<EOF
SET a 1
SET b 2
SET c 3
EOF
```

Pipelining can improve throughput by 5-10x by batching commands together. The client sends all commands at once and reads all replies at once, eliminating the wait time between commands.

### Eviction Policies

When Redis reaches `maxmemory`, it must evict keys to make room. The eviction policy determines which keys are removed.

| Policy            | Description                                            |
| ----------------- | ------------------------------------------------------ |
| `noeviction`      | Return error on writes when memory is full             |
| `allkeys-lru`     | Evict least recently used keys (recommended for cache) |
| `allkeys-lfu`     | Evict least frequently used keys                       |
| `allkeys-random`  | Evict random keys                                      |
| `volatile-lru`    | Evict LRU keys that have an expiration set             |
| `volatile-lfu`    | Evict LFU keys that have an expiration set             |
| `volatile-random` | Evict random keys that have an expiration set          |
| `volatile-ttl`    | Evict keys with the shortest TTL                       |

```conf
maxmemory 512mb
maxmemory-policy allkeys-lru
```

`allkeys-lru` is the most common policy for caching use cases. `volatile-*` policies only consider keys with an expiration set, which is useful when you want some keys to never be evicted.

### Docker: Memory-Limited Redis

```bash
docker run -d --name redis \
  -p 6379:6379 \
  --memory 512m \
  redis:7 redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
```

`--memory 512m` limits the container's total memory. `--maxmemory 256mb` limits Redis's memory usage within the container (set lower than container limit to leave room for overhead).

### Best Practices

- **Use appropriate data structures**: Hashes for objects, sorted sets for rankings, sets for unique collections.
- **Set expirations**: Always set TTL on cache keys to prevent unbounded memory growth.
- **Use SCAN instead of KEYS**: `KEYS` blocks the server; `SCAN` iterates incrementally.
- **Pipeline commands**: Reduce round trips by batching commands.
- **Use UNLINK instead of DEL**: For large keys, `UNLINK` deletes asynchronously.
- **Monitor slow queries**: Use `SLOWLOG` to find commands exceeding the threshold.
- **Avoid large keys**: Keep strings under 1 MB, collections under 10,000 elements when possible.
- **Use key namespacing**: Organize keys with colons (e.g., `user:1:profile`, `session:abc123`).
- **Disable dangerous commands** in production: `FLUSHALL`, `FLUSHDB`, `KEYS`, `CONFIG`.
- **Enable persistence**: Use RDB + AOF for production workloads.

### FastAPI Example: Pipelining & Caching

```python
@app.get("/dashboard/{user_id}")
def user_dashboard(user_id: int):
    pipe = r.pipeline()
    pipe.hgetall(f"user:{user_id}")
    pipe.get(f"balance:{user_id}")
    pipe.zrevrange(f"activity:{user_id}", 0, 4, withscores=True)
    pipe.smembers(f"user:{user_id}:tags")

    profile, balance, activity, tags = pipe.execute()

    return {
        "profile": {k.decode(): v.decode() for k, v in profile.items()} if profile else {},
        "balance": float(balance) if balance else 0,
        "recent_activity": [
            {"action": a.decode(), "score": s} for a, s in activity
        ],
        "tags": [t.decode() for t in tags],
    }

@app.post("/bulk-set")
def bulk_set(items: dict):
    pipe = r.pipeline()
    for key, value in items.items():
        pipe.set(key, value)
    results = pipe.execute()
    return {"set_count": len(results), "all_ok": all(results)}
```

`r.pipeline()` batches multiple commands into a single round trip. Instead of 4 separate requests to Redis, the dashboard fetches everything in one go. This is critical for performance when loading pages that require multiple Redis keys.
