# Redis Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Redis](#introducing-redis)
2. [Installation & Setup](#installation--setup)
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
15. [Redis with Docker](#redis-with-docker)

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

## Installation & Setup

### Install Redis

**macOS (Homebrew)**:

```bash
brew install redis
brew services start redis              # Start Redis as a background service
```

`brew services start redis` runs Redis as a launchd service that starts automatically on boot.

**Linux (Ubuntu/Debian)**:

```bash
sudo apt update
sudo apt install redis-server -y
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

`systemctl start` starts the Redis server immediately. `systemctl enable` configures Redis to start automatically on boot.

**Using Docker** (quickest way):

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

`-p 6379:6379` maps the default Redis port. This is the fastest way to get Redis running without installing anything on the host.

### Verify Installation

```bash
redis-server --version                 # Check server version
redis-cli ping                         # Should return PONG
```

`redis-cli ping` sends a PING command to the server. If Redis is running and reachable, it replies with `PONG`.

### Configuration

The main configuration file is typically located at `/etc/redis/redis.conf` (Linux) or `/usr/local/etc/redis.conf` (macOS). Key settings:

```conf
bind 127.0.0.1                         # Listen only on localhost
port 6379                              # Default port
daemonize yes                          # Run as background daemon
maxmemory 256mb                        # Limit memory usage
maxmemory-policy allkeys-lru           # Eviction policy when memory is full
```

`bind 127.0.0.1` restricts connections to localhost only, which is secure for development. `maxmemory` sets the maximum amount of RAM Redis can use. `allkeys-lru` evicts the least recently used keys when memory is full.

### Start with Custom Config

```bash
redis-server /path/to/redis.conf       # Start with specific config
redis-server --port 6380               # Start on a custom port
```

---

## Redis CLI Basics

`redis-cli` is the command-line interface for interacting with Redis. It connects to `127.0.0.1:6379` by default.

### Connecting

```bash
redis-cli                              # Connect to localhost:6379
redis-cli -h 192.168.1.10             # Connect to a remote host
redis-cli -p 6380                      # Connect to a custom port
redis-cli -a yourpassword              # Connect with password
redis-cli -n 2                         # Connect to database 2
redis-cli -u redis://user:pass@host:6379/0  # Connect via URI
```

`-h` specifies the hostname. `-p` specifies the port. `-a` provides the authentication password. `-n` selects a database number (Redis has 16 databases by default, numbered 0-15).

### Useful CLI Commands

```bash
redis-cli INFO                         # Server info and statistics
redis-cli INFO memory                  # Memory usage details
redis-cli INFO replication             # Replication status
redis-cli DBSIZE                       # Number of keys in current database
redis-cli MONITOR                      # Watch all commands in real time
redis-cli CONFIG GET maxmemory         # Get a config value
redis-cli CONFIG SET maxmemory 512mb   # Set a config value at runtime
redis-cli SLOWLOG GET 10               # Show 10 slowest queries
redis-cli CLIENT LIST                  # List all connected clients
```

`MONITOR` streams every command Redis receives in real time, useful for debugging but expensive in production. `SLOWLOG` shows commands that exceeded the configured slow threshold (default 10ms). `CONFIG SET` changes settings at runtime without restarting.

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

---

## Pub/Sub Messaging

Redis Pub/Sub provides a publish/subscribe messaging system. Publishers send messages to channels, and subscribers receive messages from channels they are subscribed to. Messages are fire-and-forget — if no subscriber is listening, the message is lost.

### Subscribing

```bash
SUBSCRIBE news                         # Subscribe to "news" channel
SUBSCRIBE news sports weather          # Subscribe to multiple channels
PSUBSCRIBE news:*                      # Subscribe to pattern (all news:* channels)
```

`SUBSCRIBE` blocks the connection and waits for messages. Once subscribed, the connection can only run subscribe-related commands. `PSUBSCRIBE` uses glob-style patterns to match multiple channels.

### Publishing

```bash
PUBLISH news "Breaking news!"          # Publish message → returns subscriber count
PUBLISH news:sports "Goal scored!"     # Publish to a specific sub-channel
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

| Feature          | RDB                       | AOF                        |
| ---------------- | ------------------------- | -------------------------- |
| Data Loss Risk   | Data between snapshots    | At most 1 second (default) |
| File Size        | Compact                   | Larger (full command log)  |
| Restart Speed    | Faster (load binary)      | Slower (replay commands)   |
| Write Impact     | Fork + write (periodic)   | Continuous writes          |
| Best For         | Backups, disaster recovery | Durability, minimal loss   |

### Recommended: Both RDB + AOF

```conf
appendonly yes
save 900 1
save 300 10
save 60 10000
```

Using both gives you the best of both worlds: RDB for fast restarts and backups, AOF for minimal data loss. If both are present at startup, Redis loads the AOF because it is typically more complete.

---

## Replication

Redis replication creates copies (replicas) of a master Redis instance. Replicas maintain an exact copy of the master's data and can serve read requests, distributing the read load.

### Setting Up Replication

On the replica server, add to `redis.conf`:

```conf
replicaof 192.168.1.10 6379            # Master host and port
masterauth yourpassword                # Master password (if set)
replica-read-only yes                  # Replica accepts only reads (default)
```

Or at runtime:

```bash
REPLICAOF 192.168.1.10 6379            # Make this instance a replica
REPLICAOF NO ONE                       # Promote replica to master
INFO replication                       # Check replication status
```

`REPLICAOF` configures the current instance as a replica of the specified master. `REPLICAOF NO ONE` promotes the replica to an independent master. `replica-read-only yes` prevents writes to the replica, which is the default and recommended setting.

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

---

## Redis Sentinel

Sentinel provides high availability for Redis. It monitors master and replica instances, detects failures, and performs automatic failover.

### Sentinel Configuration

Create `sentinel.conf`:

```conf
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster yourpassword
```

`sentinel monitor` defines the master to monitor; the last number (`2`) is the quorum — the minimum number of Sentinels that must agree a master is down before failover. `down-after-milliseconds` is how long a master must be unreachable before it is considered down. `failover-timeout` sets the maximum time for the failover process. `parallel-syncs` controls how many replicas sync to the new master simultaneously.

### Starting Sentinel

```bash
redis-sentinel /path/to/sentinel.conf
# or
redis-server /path/to/sentinel.conf --sentinel
```

### Sentinel Commands

```bash
redis-cli -p 26379                     # Connect to Sentinel (default port)
SENTINEL masters                       # List all monitored masters
SENTINEL replicas mymaster             # List replicas of a master
SENTINEL get-master-addr-by-name mymaster  # Get current master address
SENTINEL failover mymaster             # Force a manual failover
```

In production, run at least 3 Sentinel instances on separate machines to form a reliable quorum. Clients connect to Sentinel first to discover the current master, then connect to the master directly.

---

## Redis Cluster

Redis Cluster provides automatic sharding across multiple Redis nodes. Data is distributed using hash slots — the key space is divided into 16,384 slots, and each node is responsible for a subset of slots.

### How Cluster Works

- The keyspace is divided into **16,384 hash slots**.
- Each master node owns a subset of slots.
- The hash slot for a key is computed as `CRC16(key) mod 16384`.
- Each master can have one or more replicas for failover.
- Clients are redirected to the correct node if they send a command to the wrong one.

### Setting Up a Cluster

Create configuration for each node:

```conf
# redis-7000.conf
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes
```

Start each node:

```bash
redis-server redis-7000.conf
redis-server redis-7001.conf
redis-server redis-7002.conf
redis-server redis-7003.conf
redis-server redis-7004.conf
redis-server redis-7005.conf
```

Create the cluster:

```bash
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

`--cluster-replicas 1` assigns one replica per master. With 6 nodes and 1 replica each, you get 3 masters and 3 replicas.

### Cluster Commands

```bash
redis-cli -c -p 7000                  # Connect in cluster mode
CLUSTER INFO                           # Cluster status
CLUSTER NODES                          # List all nodes and their slots
CLUSTER SLOTS                          # Slot-to-node mapping
CLUSTER KEYSLOT mykey                  # Which slot does "mykey" belong to?

redis-cli --cluster check 127.0.0.1:7000     # Health check
redis-cli --cluster reshard 127.0.0.1:7000   # Redistribute slots
redis-cli --cluster add-node new:7006 existing:7000  # Add a node
redis-cli --cluster del-node 127.0.0.1:7000 <node-id> # Remove a node
```

`-c` enables cluster mode in `redis-cli`, which automatically follows `MOVED` and `ASK` redirections. `reshard` moves hash slots between nodes for rebalancing. `add-node` adds a new empty node to the cluster.

### Hash Tags

```bash
SET {user:1}.name "Bidur"
SET {user:1}.email "bidur@example.com"
```

Hash tags (`{...}`) ensure that keys with the same tag hash to the same slot, so they can be accessed together in multi-key operations. Only the substring inside `{}` is used for hashing. This is required for multi-key commands like `MGET` in a cluster.

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
redis-cli
AUTH yourpassword                      # Authenticate after connecting
redis-cli -a yourpassword              # Authenticate on connect
```

`requirepass` sets a single server-wide password. Redis 6+ introduced ACLs (Access Control Lists) for more granular permissions with username-based authentication.

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

`bind 127.0.0.1` prevents Redis from accepting connections from external networks. `protected-mode yes` blocks connections from non-loopback interfaces if no password is set. `rename-command "" ` effectively disables a command by renaming it to an empty string.

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

| Type       | Small Encoding | Large Encoding | Threshold                |
| ---------- | -------------- | -------------- | ------------------------ |
| String     | embstr / int   | raw            | 44 bytes                 |
| List       | listpack       | quicklist      | 128 elements / 64 bytes  |
| Set        | listpack       | hashtable      | 128 elements / 64 bytes  |
| Sorted Set | listpack       | skiplist        | 128 elements / 64 bytes  |
| Hash       | listpack       | hashtable      | 128 elements / 64 bytes  |

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

| Policy            | Description                                           |
| ----------------- | ----------------------------------------------------- |
| `noeviction`      | Return error on writes when memory is full            |
| `allkeys-lru`     | Evict least recently used keys (recommended for cache)|
| `allkeys-lfu`     | Evict least frequently used keys                      |
| `allkeys-random`  | Evict random keys                                     |
| `volatile-lru`    | Evict LRU keys that have an expiration set            |
| `volatile-lfu`    | Evict LFU keys that have an expiration set            |
| `volatile-random` | Evict random keys that have an expiration set         |
| `volatile-ttl`    | Evict keys with the shortest TTL                      |

```conf
maxmemory 512mb
maxmemory-policy allkeys-lru
```

`allkeys-lru` is the most common policy for caching use cases. `volatile-*` policies only consider keys with an expiration set, which is useful when you want some keys to never be evicted.

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

---

## Redis with Docker

### Running Redis in Docker

```bash
docker run -d --name redis -p 6379:6379 redis:7
docker run -d --name redis -p 6379:6379 redis:7 redis-server --requirepass yourpassword
```

The second command starts Redis with password authentication by overriding the default command.

### Redis with Persistent Storage

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  redis:7 redis-server --appendonly yes
```

`-v redis-data:/data` mounts a named volume to `/data`, which is where Redis stores its dump files. `--appendonly yes` enables AOF persistence.

### Redis with Custom Config

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redis-data:/data \
  -v ./redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7 redis-server /usr/local/etc/redis/redis.conf
```

This bind-mounts a custom `redis.conf` from the host into the container and tells Redis to use it.

### Docker Compose: App + Redis

```yaml
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
    command: redis-server --appendonly yes --requirepass yourpassword

volumes:
  redis-data:
```

`depends_on` ensures Redis starts before the application. The `api` service connects to Redis using `redis` as the hostname (the service name) because Docker Compose creates a shared network. `command` overrides the default container command to enable persistence and set a password.

### Connecting from Application

```bash
# Connect to Redis running in Docker
redis-cli -h 127.0.0.1 -p 6379 -a yourpassword

# From another container on the same network
redis-cli -h redis -p 6379 -a yourpassword
```

### Redis Cluster with Docker Compose

```yaml
services:
  redis-node-1:
    image: redis:7
    ports:
      - "7000:7000"
      - "17000:17000"
    volumes:
      - ./redis-7000.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-2:
    image: redis:7
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - ./redis-7001.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-3:
    image: redis:7
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - ./redis-7002.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
```

Each node gets its own configuration file with `cluster-enabled yes` and its respective port. After all nodes are running, use `redis-cli --cluster create` to form the cluster.
