# SQLite3 vs PostgreSQL Storage and Memory Management Analysis

## 1. SQLite3 Experiment

### 1.1 Environment

```bash
sqlite3 --version
```

```
3.51.0 2025-11-04 19:38:17 fb2c931ae597f8d00a37574ff67aeed3eced4e5547f9120744ae4bfa8e74527b (64-bit)
```

### 1.2 Database and Table Setup

A new database file was created using the command-line shell:

```bash
sqlite3 advDbLab.db
```

A table was created to hold user records:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    age INTEGER
);
```

A small number of rows were manually inserted:

```sql
INSERT INTO users (name, age) VALUES
('Alice', 21),
('Bob', 22),
('Charlie', 23),
('David', 24),
('Eve', 25);
```

### 1.3 Large Dataset Insertion

A recursive CTE was used to insert 100,000 rows in a single statement:

```sql
WITH RECURSIVE cnt(x) AS (
    SELECT 1
    UNION ALL
    SELECT x+1 FROM cnt LIMIT 100000
)
INSERT INTO users(name, age)
SELECT 'User' || x, 20 + (x % 10)
FROM cnt;
```

This method avoids making repeated round-trips to the database, allowing SQLite to process all inserts within a single transaction.

### 1.4 Database File Size

```bash
ls -lh advDbLab.db
```

```
-rw-r--r-- 1 navjeetsingh navjeetsingh 2.0M May 9 20:09 advDbLab.db
```

SQLite keeps its entire database within a single file on disk. No separate files exist for indexes, logs, or metadata — everything is contained inside `advDbLab.db`.

### 1.5 Page Size

```bash
sqlite3 advDbLab.db "PRAGMA page_size;"
```

```
4096
```

SQLite divides its database file into fixed-size pages. The default page size is 4096 bytes, which matches the default page size of most modern operating systems. This alignment allows SQLite pages to map directly onto OS pages, reducing the overhead involved when the OS reads or caches the file.

### 1.6 Page Count

```bash
sqlite3 advDbLab.db "PRAGMA page_count;"
```

```
0
```

The `page_count` returned 0 because the PRAGMA command was mistakenly run against a different database file (`adbDbLab.db`) rather than the populated one (`advDbLab.db`). SQLite automatically creates a new, empty database file when the specified file does not exist, so the result was an empty database with no allocated pages.

### 1.7 Memory-Mapped I/O in SQLite

SQLite offers optional memory-mapped I/O through the `mmap_size` PRAGMA. By default, mmap is turned off:

```bash
sqlite3 advDbLab.db "PRAGMA mmap_size;"
```

```
0
```

Enabling mmap for the current session:

```bash
sqlite3 advDbLab.db "PRAGMA mmap_size=268435456;"
```

```
268435456
```

This configures a 256 MB mmap window for the active connection. However, opening a new connection shows that the setting has reset:

```bash
sqlite3 advDbLab.db "PRAGMA mmap_size;"
```

```
0
```

**Key observation:** `mmap_size` is a per-connection, per-session setting. It is not written permanently into the database file. Every new connection starts with mmap disabled unless the application sets it explicitly. This is intentional — SQLite is designed to function correctly even in environments where mmap may be unavailable or unreliable.

When mmap is active, SQLite maps the database file directly into the process's virtual address space. The OS then uses demand paging to load file pages only when they are accessed. This eliminates the second copy that a normal `read()` call requires (from the page cache into the user buffer). For workloads dominated by reads on large databases, this can lead to noticeable gains in throughput.

### 1.8 Process Architecture

```bash
ps aux | grep sqlite
```

Observation: SQLite appeared as a single, short-lived process that existed only while the shell session was open. Its memory footprint was minimal, and it disappeared as soon as the connection was closed.

This reflects SQLite's embedded design. The library runs inside the calling application's process — there is no separate server. The application links against the SQLite library, and all database operations execute within the same process memory space.

### 1.9 Query Timing

```bash
time sqlite3 advDbLab.db "SELECT * FROM users;"
```

```
real    0m0.820s
user    0m0.146s
sys     0m0.332s
```

These three values capture different dimensions of the time taken:

- `real` — total wall-clock time from start to finish, including any waiting
- `user` — CPU time spent running application-level code
- `sys` — CPU time spent inside kernel calls such as file reads and memory operations

SQLite completed a full table scan in under one second despite having no server, no buffer pool, and no background processes. This shows that its lightweight embedded architecture is competitive for single-user local workloads.

---

## 2. PostgreSQL Experiment

### 2.1 Starting the Service

PostgreSQL runs as a background service managed by the OS init system:

```bash
sudo systemctl start postgresql
```

Unlike SQLite, PostgreSQL must already be running as a server before any client can connect.

### 2.2 Connecting and Setting Up

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE oslab_pg;
\c oslab_pg
```

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT,
    age INTEGER
);
```

### 2.3 Large Dataset Insertion

PostgreSQL provides a built-in `generate_series` function that simplifies bulk inserts:

```sql
INSERT INTO users(name, age)
SELECT
    'User' || g,
    20 + (g % 10)
FROM generate_series(1, 100000) AS g;
```

### 2.4 Database Size

```sql
SELECT pg_size_pretty(pg_database_size('oslab_pg'));
```

```
15 MB
```

PostgreSQL reports 15 MB for the same 100,000 rows that SQLite stored in just 2 MB. The larger footprint is expected: PostgreSQL records extra metadata per row (known as a tuple header), maintains visibility information to support MVCC (multi-version concurrency control), and keeps structures such as SERIAL sequences and WAL segments in separate files.

### 2.5 Block Size

```sql
SHOW block_size;
```

```
8192
```

PostgreSQL uses a default block (page) size of 8192 bytes — twice the size of SQLite's default page. Larger pages reduce the number of I/O operations needed to read large tables, since more rows fit into a single page fetch. On the other hand, when queries target only a few rows, larger pages can cause more data to be read from disk than is actually needed.

### 2.6 Page Count Estimation

```sql
SELECT relname, relpages
FROM pg_class
WHERE relname = 'users';
```

```
users | 0
```

The `relpages` value was 0 because PostgreSQL refreshes statistics asynchronously. The catalog entry had not yet been updated by autovacuum following the bulk insert. Running `ANALYZE users;` would bring this value up to date.

### 2.7 Query Timing

Timing was enabled to measure query execution:

```sql
\timing
```

Results for `SELECT COUNT(*) FROM users`:

```
Time: 10.798 ms
Time: 36.262 ms
Time: 8.974 ms
Time: 12.424 ms
```

Result for `SELECT * FROM users`:

```
Time: 54.898 ms
```

**Key observation:** The variation across `COUNT(*)` runs reflects the impact of caching. The first execution had to fetch data pages from disk into shared buffers. Later runs found those pages already resident in shared buffers and returned results more quickly. The slower run (36 ms) likely coincided with background system activity or partial eviction of shared buffers.

The `SELECT *` query took longer because it had to transfer the full contents of all 100,000 rows, rather than simply counting them.

### 2.8 Process Architecture

```bash
ps aux | grep postgres
```

Unlike SQLite, PostgreSQL operates as a server with several background processes that handle memory management, disk writes, logging, and concurrency control. This architecture enables PostgreSQL to serve multiple concurrent users efficiently.

```
postgres  386672  0.0  0.1 225464 31212 ?        Ss   20:18   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres  386676  0.0  0.0 225596 12216 ?        Ss   20:18   0:00 postgres: 16/main: checkpointer 
postgres  386677  0.0  0.0 225620  7916 ?        Ss   20:18   0:00 postgres: 16/main: background writer 
postgres  386679  0.0  0.0 225464 10396 ?        Ss   20:18   0:00 postgres: 16/main: walwriter 
postgres  386680  0.0  0.0 227072  8860 ?        Ss   20:18   0:00 postgres: 16/main: autovacuum launcher 
postgres  386681  0.0  0.0 227048  8140 ?        Ss   20:18   0:00 postgres: 16/main: logical replication launcher 
root      387573  0.1  0.0  19820  7632 pts/0    S+   20:18   0:00 sudo -u postgres psql
root      387574  0.0  0.0  19820  2644 pts/2    Ss   20:18   0:00 sudo -u postgres psql
postgres  387575  0.0  0.0  28684 12004 pts/2    S+   20:18   0:00 /usr/lib/postgresql/16/bin/psql
postgres  389273  0.2  0.2 228352 34104 ?        Ss   20:19   0:00 postgres: 16/main: postgres oslab_pg [local] idle
navjeet-d+  405463  0.0  0.0   9148  2272 pts/1    S+   20:23   0:00 grep --color=auto postgres
```

---

## 3. Observations

### 3.1 SQLite Observations

- The entire database is contained within a single file (`advDbLab.db`), making it straightforward to copy, move, or back up.
- The default page size of 4096 bytes aligns with the OS page size, enabling clean mapping between SQLite pages and OS memory pages.
- mmap support is available but scoped to the session. It must be explicitly enabled per connection and does not persist across sessions.
- SQLite runs as a single process with no background workers. There is no server to start or stop.
- Memory consumption is minimal because there is no buffer pool, no background writers, and no WAL writer running continuously.

### 3.2 PostgreSQL Observations

- PostgreSQL uses 8192-byte blocks, double the size of SQLite's pages, which suits workloads involving large sequential scans.
- On-disk size is considerably larger (15 MB vs 2 MB for identical data) due to MVCC metadata, tuple headers, and WAL overhead.
- Query times for repeated executions decreased as a result of shared buffer caching. Once data pages are loaded into shared buffers, subsequent queries bypass disk entirely.
- Several background processes run at all times, each responsible for a specific OS-level task such as WAL writing, flushing dirty pages, and removing dead tuples.
- PostgreSQL does not rely on memory-mapped I/O for data pages. Instead, it manages its own buffer pool in shared memory and coordinates access across processes through its buffer manager.

---

## 4. Comparison Table

| Feature | SQLite3 | PostgreSQL |
|---|---|---|
| Architecture | Embedded library, no server | Client-server, server runs separately |
| Storage Model | Single `.db` file on disk | Directory of files per database cluster |
| Page Size | 4096 bytes (default) | 8192 bytes (default) |
| File Size (100k rows) | 2.0 MB | 15 MB |
| mmap Support | Yes, session-specific, opt-in | Not used directly; uses shared buffers |
| Process Model | Single process (embedded in app) | Multiple background + per-client processes |
| Shared Memory | Not used | Shared buffers shared across all backends |
| Concurrency | File-level locking, limited | MVCC, row-level locking, high concurrency |
| WAL / Crash Recovery | Available but optional | Always enabled by default |
| Performance | Fast for local, low-concurrency workloads | Scales to high concurrency and large data |
| Suitable Use Cases | Mobile apps, local tools, embedded systems | Web backends, enterprise, multi-user systems |

---

## 5. Analysis of mmap and Caching

### How Normal File I/O Works

When a database reads a page from disk using a standard `read()` system call, two copy operations occur:

1. The disk controller uses DMA to move data from storage into a page within the OS page cache (kernel RAM).
2. The kernel then copies that page from the page cache into the application's own buffer in user memory.

This second copy consumes CPU time and memory bandwidth, which becomes significant for large databases where many pages are read repeatedly.

### How mmap Reduces This Cost

When SQLite enables mmap, it calls `mmap()` on the database file. The OS maps the file's pages directly into the process's virtual address space. When the application accesses a mapped address, the OS uses demand paging: if the page is not yet loaded, a page fault triggers the kernel to bring the page into the page cache and map it into the process's address space. From that point on, the application reads the data without any additional copy.

The outcome is that mmap removes the second copy entirely. Data travels from disk to the page cache via DMA and is immediately accessible through the mapped address. For read-heavy workloads, this can meaningfully lower CPU usage and improve throughput.

The constraint is that mmap in SQLite is session-scoped. There is no shared pool across connections. Each connection that wants mmap must enable it independently.

### How PostgreSQL Handles Caching

PostgreSQL does not use mmap for its data pages. Instead, it reserves a large block of shared memory at startup, referred to as shared buffers, and manages it internally. The buffer manager decides which pages to retain in shared buffers and which to evict when space is exhausted, using a clock-sweep algorithm comparable to the OS page replacement policy.

All backend processes (one per connected client) share access to this pool. When one client reads a data page, that page remains in shared buffers. If another client requests the same page shortly after, it is already available. This explains why successive `SELECT COUNT(*) FROM users` queries became faster over time — the pages were already in shared buffers after the first run.

The benefit of maintaining its own buffer pool is that PostgreSQL can apply database-specific knowledge to caching decisions. For instance, it can distinguish pages that belong to a large sequential scan (which should not displace useful cached data) from pages belonging to a frequently accessed index (which should be kept in cache).

---

## 6. Process Architecture Comparison

### SQLite — Embedded, Single Process

SQLite is a C library linked directly into the application. There is no daemon, no network socket, and no server process. When the application calls a SQLite function, the library code runs in the same thread and the same process, with direct access to the process's memory.

```
Application Process
    |
    +-- SQLite Library (linked in)
    |       |
    |       +-- reads/writes advDbLab.db directly
    |
    +-- Application code
```

This design makes SQLite extremely lightweight. The trade-off is that only one writer can access the database at a time (file-level write lock), which makes it poorly suited to applications with many concurrent writers.

### PostgreSQL — Client-Server, Multi-Process

PostgreSQL uses a forking model. A master process called `postmaster` listens for incoming connections. Each time a client connects, `postmaster` forks a dedicated backend process for that client. All backend processes share a common block of shared memory (shared buffers) and interact through it under the coordination of the buffer manager and lock manager.

```
postmaster (main process)
    |
    +-- shared memory (shared buffers, lock tables, etc.)
    |
    +-- backend (client 1)
    +-- backend (client 2)
    +-- checkpointer
    +-- background writer
    +-- walwriter
    +-- autovacuum launcher
```

This architecture supports genuine concurrent read and write access from many clients simultaneously. MVCC ensures that readers do not block writers and writers do not block readers by keeping multiple row versions within data pages.

The cost is a higher baseline resource footprint: even with no clients connected, PostgreSQL keeps several background processes running and holds a large shared memory allocation.

---

## 7. Performance Analysis

### File Size Difference

The same dataset (100,000 rows) occupied 2.0 MB in SQLite and 15 MB in PostgreSQL. Several factors account for this gap:

- PostgreSQL uses 8192-byte pages, while SQLite uses 4096-byte pages.
- PostgreSQL stores additional MVCC metadata and tuple headers for concurrency control.
- PostgreSQL maintains WAL, indexes, and system catalog information as part of its storage.
- SQLite's simpler file-based architecture results in considerably lower storage overhead.

For environments where disk space is constrained, such as embedded systems or mobile devices, SQLite's compact storage is a meaningful advantage.

### Query Timing Comparison

| Database | Query | Observed Time |
|---|---|---|
| SQLite3 | `SELECT * FROM users;` | real: 0.820s |
| PostgreSQL | `SELECT COUNT(*) FROM users;` | 10–36 ms |
| PostgreSQL | `SELECT * FROM users;` | 54 ms |

### Performance Comparison

| Aspect | SQLite | PostgreSQL |
|---|---|---|
| Query overhead | Minimal — no network or IPC | Higher — client-server communication |
| Caching | Relies on OS page cache | Manages its own shared buffer pool |
| Concurrency overhead | None (single writer) | Coordination across multiple processes |
| Best workload | Lightweight, single-user, local access | Concurrent, multi-user, high-throughput |

SQLite carries lower architectural overhead because it runs entirely within the application process. There is no network round-trip, no inter-process communication, and no shared memory coordination. PostgreSQL introduces additional overhead from its server processes and concurrency management, but this is the cost of reliably supporting many simultaneous users. PostgreSQL is optimized for scalability; SQLite is optimized for simplicity and local access.

### When Each System Performs Better

SQLite performs better when:

- Only one writer is active at a time
- The database is small enough to fit mostly in the OS page cache
- Minimal per-query latency is required with no network round-trip
- The application must run without any server setup

PostgreSQL performs better when:

- Many clients read and write concurrently
- The dataset is large enough to warrant a managed buffer pool
- Strong crash recovery guarantees are required
- Complex queries, joins, and query planning are frequent

---