# Distributed Database Environment using Docker + MariaDB (FEDERATED Engine)

**Author:** Antu Chowdhury

**Topic:** Building a Simple Distributed Database using Data Fragmentation

**Tools Used:** Docker Desktop, MariaDB 10.11 (via Docker), FEDERATED/FederatedX Storage Engine

> \*\*Note:\*\* All passwords in this report (`<root\_password>`, `<db\_password>`) are placeholders. Replace them with your own values before running any command — do not use literal angle-bracket text as a real password.

\---

## 1\. Objective

To design and implement a simple **distributed database environment** demonstrating **horizontal data fragmentation** across two independent database nodes, queried transparently as a single logical database — without the client needing to know where each piece of data physically resides.

\---

## 2\. Core Concept: What "Distributed Database" Means Here

A distributed database is not simply "two databases that talk to each other" — the defining property is **transparency**. A user or application should be able to run one query and get correct results, regardless of which physical node actually stores each row.

There are two common ways people implement something they *call* "distributed":

|Approach|What it means|What we built|
|-|-|-|
|**Replication**|Every node has a *full copy* of the same data (used for redundancy/availability)|❌ Not this|
|**Fragmentation**|Data is *split* across nodes — each node holds a different slice|✅ This|

We implemented **horizontal fragmentation**: rows of a `customers` table were split by region — Dhaka customers physically live on Node 1 (`pc1`), Chittagong customers physically live on Node 2 (`pc2`) — then unified into one queryable view.

### Key DBMS terms demonstrated

* **Fragmentation** — splitting a table's rows (horizontal) or columns (vertical) across nodes.
* **Transparency** — the client queries `customers\_global` and doesn't need to know 2 of the rows come from a completely different server.
* **FEDERATED / FederatedX Engine** — a MariaDB storage engine that creates a *local proxy table* pointing at a table on a remote server. Reading/writing to the proxy table transparently forwards the operation to the remote server via MySQL's client protocol (`libmysql`).

\---

## 3\. Architecture

```
                 ┌─────────────────────────────┐
                 │     Docker Network:          │
                 │     distdb-net (bridge)      │
                 │                               │
   ┌─────────────┴──────────┐    ┌───────────────┴─────────────┐
   │   pc1 (MariaDB)         │    │   pc2 (MariaDB)              │
   │   Host port: 3316       │    │   Host port: 3307             │
   │   Internal port: 3306   │    │   Internal port: 3306         │
   │                          │    │                               │
   │  customers\_dhaka (real) │    │  customers\_ctg (real)         │
   │  customers\_ctg\_link ────┼────┼─► points to customers\_ctg     │
   │   (FEDERATED proxy)     │    │                               │
   │                          │    │                               │
   │  customers\_global (VIEW)│    │                               │
   │  = customers\_dhaka      │    │                               │
   │    UNION ALL             │    │                               │
   │    customers\_ctg\_link    │    │                               │
   └──────────────────────────┘    └───────────────────────────────┘
```

We chose **Docker containers** instead of two physical PCs on a LAN because Docker's internal DNS (container name resolution: `pc1`, `pc2`) eliminates the need for IP address management, port forwarding across a real router, and Windows Firewall configuration — all of which caused significant friction when first attempting this on real hardware over Wi-Fi/LAN.

\---

## 4\. Full Setup — Step by Step

### 4.1 Prerequisites

* Docker Desktop installed and running
* **Hardware virtualization enabled in BIOS** (Intel VT-x / AMD-V) — required for Docker's WSL2 backend to run at all

### 4.2 Project folder structure

```
distdb-project/
├── docker-compose.yml
├── pc1-config/
│   └── federated.cnf
└── pc2-config/
    └── federated.cnf
```

### 4.3 `docker-compose.yml`

```yaml
version: '3.8'
services:
  pc1:
    image: mariadb:10.11
    container\_name: pc1
    environment:
      MYSQL\_ROOT\_PASSWORD: <root\_password>
      MYSQL\_DATABASE: company\_db
      MYSQL\_USER: fed\_user
      MYSQL\_PASSWORD: <db\_password>
    ports:
      - "3316:3306"
    volumes:
      - ./pc1-config/federated.cnf:/etc/mysql/conf.d/federated.cnf
    networks:
      - distdb-net

  pc2:
    image: mariadb:10.11
    container\_name: pc2
    environment:
      MYSQL\_ROOT\_PASSWORD: <root\_password>
      MYSQL\_DATABASE: company\_db
      MYSQL\_USER: fed\_user
      MYSQL\_PASSWORD: <db\_password>
    ports:
      - "3307:3306"
    volumes:
      - ./pc2-config/federated.cnf:/etc/mysql/conf.d/federated.cnf
    networks:
      - distdb-net

networks:
  distdb-net:
    driver: bridge
```

> \*\*Note:\*\* Host ports (left side of `3316:3306`) had to be changed from the default `3306` because a local XAMPP MySQL install was already using it on the host machine. The \*\*internal\*\* port (right side, `3306`) stays standard — that's what containers use to reach each other, and it's what matters for the FEDERATED connection string.

### 4.4 Launch containers

```
docker-compose up -d
docker ps          # confirm both pc1 and pc2 show "Up"
```

### 4.5 Enable the FEDERATED engine (must be done via SQL, not just the config file)

On **each** container:

```
docker exec -it pc1 mysql -u root -p<root\_password>
```

```sql
INSTALL SONAME 'ha\_federatedx';
SHOW ENGINES;   -- confirm FEDERATED shows Support: YES
```

Repeat for `pc2`.

### 4.6 Create the data fragments

**On pc1** (`docker exec -it pc1 mysql -u fed\_user -p<db\_password> company\_db`):

```sql
CREATE TABLE customers\_dhaka (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
);
INSERT INTO customers\_dhaka VALUES
(1, 'Rafiq Islam', 'Dhaka', 'rafiq@example.com'),
(2, 'Nusrat Jahan', 'Dhaka', 'nusrat@example.com');
```

**On pc2** (`docker exec -it pc2 mysql -u fed\_user -p<db\_password> company\_db`):

```sql
CREATE TABLE customers\_ctg (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
);
INSERT INTO customers\_ctg VALUES
(3, 'Farhan Kabir', 'Chittagong', 'farhan@example.com'),
(4, 'Sabrina Akter', 'Chittagong', 'sabrina@example.com');
```

### 4.7 Create the FEDERATED link (on pc1, pointing at pc2)

```sql
CREATE TABLE customers\_ctg\_link (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
) ENGINE=FEDERATED
CONNECTION='mysql://fed\_user:<db\_password>@pc2:3306/company\_db/customers\_ctg';
```

Note the connection string uses the **container name** (`pc2`) and **internal port** (`3306`) — not the host-mapped port.

### 4.8 Build the unified view (on pc1)

```sql
CREATE VIEW customers\_global AS
SELECT \* FROM customers\_dhaka
UNION ALL
SELECT \* FROM customers\_ctg\_link;
```

### 4.9 Verify

```sql
SELECT \* FROM customers\_global;
```

Expected: all 4 rows — 2 stored locally on pc1, 2 fetched live from pc2 in real time.

\---

## 5\. Orders Fragment — Full Setup \& Cross-Node JOIN Demo

This second fragment is what elevates the demo from a simple `UNION` of two tables into a genuine test of distributed querying: a **JOIN across two physically separate nodes**. `customers` data and `orders` data both stay fragmented (Dhaka vs Chittagong), and we prove a query engine can still relate them correctly as if they lived in one database.

### 5.1 Create the orders fragment on pc1 (Dhaka orders)

```
docker exec -it pc1 mysql -u fed\_user -p<db\_password> company\_db
```

```sql
CREATE TABLE orders\_dhaka (
  order\_id INT PRIMARY KEY,
  customer\_id INT,
  amount DECIMAL(10,2)
);

INSERT INTO orders\_dhaka VALUES
(101, 1, 1500.00),
(102, 2, 2300.00);
```

(`customer\_id` 1 and 2 refer to Rafiq and Nusrat, who live in `customers\_dhaka`.)

### 5.2 Create the orders fragment on pc2 (Chittagong orders)

```
docker exec -it pc2 mysql -u fed\_user -p<db\_password> company\_db
```

```sql
CREATE TABLE orders\_ctg (
  order\_id INT PRIMARY KEY,
  customer\_id INT,
  amount DECIMAL(10,2)
);

INSERT INTO orders\_ctg VALUES
(201, 3, 1800.00),
(202, 4, 950.00);
```

(`customer\_id` 3 and 4 refer to Farhan and Sabrina, who live in `customers\_ctg`.)

### 5.3 Link pc2's orders into pc1 via FEDERATED

Back on pc1:

```sql
CREATE TABLE orders\_ctg\_link (
  order\_id INT PRIMARY KEY,
  customer\_id INT,
  amount DECIMAL(10,2)
) ENGINE=FEDERATED
CONNECTION='mysql://fed\_user:<db\_password>@pc2:3306/company\_db/orders\_ctg';
```

Test the link on its own first:

```sql
SELECT \* FROM orders\_ctg\_link;
```

Expected: the two Chittagong orders (201, 202), fetched live from pc2.

### 5.4 Build the unified orders view

Still on pc1:

```sql
CREATE VIEW orders\_global AS
SELECT \* FROM orders\_dhaka
UNION ALL
SELECT \* FROM orders\_ctg\_link;
```

Test it:

```sql
SELECT \* FROM orders\_global;
```

Expected: all 4 orders (101, 102, 201, 202) — 2 physically local, 2 pulled from pc2.

### 5.5 The cross-node JOIN — customers + orders together

This is the real test: joining `customers\_global` (itself already spanning 2 nodes) with `orders\_global` (also spanning 2 nodes):

```sql
SELECT c.name, c.region, o.order\_id, o.amount
FROM customers\_global c
JOIN orders\_global o ON c.id = o.customer\_id
ORDER BY c.id;
```

Expected output:

```
+---------------+-------------+----------+---------+
| name          | region      | order\_id | amount  |
+---------------+-------------+----------+---------+
| Rafiq Islam   | Dhaka       | 101      | 1500.00 |
| Nusrat Jahan  | Dhaka       | 102      | 2300.00 |
| Farhan Kabir  | Chittagong  | 201      | 1800.00 |
| Sabrina Akter | Chittagong  | 202      | 950.00  |
+---------------+-------------+----------+---------+
```

Every row here is correctly matched even though the underlying `customer` and `order` records for each region live on physically different containers.

### 5.6 Aggregate query across fragments

```sql
SELECT region, SUM(amount) AS total\_sales
FROM customers\_global c
JOIN orders\_global o ON c.id = o.customer\_id
GROUP BY region;
```

Expected output:

```
+-------------+-------------+
| region      | total\_sales |
+-------------+-------------+
| Dhaka       | 3800.00     |
| Chittagong  | 2750.00     |
+-------------+-------------+
```

This shows region-level totals computed from data that was never physically stored together — the aggregation happens transparently across nodes.

### 5.7 Write operations through the FEDERATED link (proving it's bidirectional)

```sql
-- INSERT: lands on pc2, not pc1
INSERT INTO orders\_ctg\_link VALUES (203, 4, 500.00);

-- UPDATE: modifies the row on pc2
UPDATE orders\_ctg\_link SET amount = 600.00 WHERE order\_id = 203;

-- DELETE: removes the row from pc2
DELETE FROM orders\_ctg\_link WHERE order\_id = 203;
```

Verify any of these directly on pc2 to confirm the write actually happened remotely:

```
docker exec -it pc2 mysql -u fed\_user -p<db\_password> company\_db -e "SELECT \* FROM orders\_ctg;"
```

### 5.8 Fault-tolerance check (demonstrating a real limitation)

```
docker stop pc2
```

Then on pc1:

```sql
SELECT \* FROM orders\_global;
```

This will **error out** on the `orders\_ctg\_link` portion — proving FEDERATED has no replication or failover; if the remote node is down, that fragment is simply unavailable. Restart pc2 afterward:

```
docker start pc2
```

### 5.9 Query plan inspection

```sql
EXPLAIN SELECT c.name, o.amount
FROM customers\_global c
JOIN orders\_global o ON c.id = o.customer\_id;
```

Shows how the optimizer treats the FEDERATED-backed portions differently from purely local tables — useful talking point on distributed query cost.

\---

## 6\. Errors Encountered \& Solutions (Full Troubleshooting Log)

### 6.1 `Got error 176 "Read page with wrong checksum" from storage engine Aria`

* **Context:** Happened on the original XAMPP MySQL (before moving to Docker).
* **Cause:** Corrupted Aria storage engine log files, almost always from an unclean shutdown (crash, force-close, power loss).
* **Solution:**

  1. Back up `C:\\xampp\\mysql\\data`
  2. Stop MySQL
  3. Rename `aria\_log\_control` and all `aria\_log.000000X` files to `.bak`
  4. Restart MySQL — fresh logs auto-generate
  5. If needed, run `mysqlcheck -u user -p --auto-repair --all-databases`

### 6.2 `Virtualization support not detected` (Docker Desktop startup error)

* **Cause:** Hardware virtualization (Intel VT-x) disabled in BIOS.
* **Solution:**

  1. Task Manager → Performance → CPU → check if "Virtualization" says Enabled/Disabled
  2. If Disabled: restart → enter BIOS (Lenovo: F1 or F2) → **Security tab** → find **Intel Virtualization Technology (VT-x)** → set to **Enabled** → Save \& Exit (F10)
  3. Also enable Windows features: `optionalfeatures` → check **Virtual Machine Platform** and **Windows Subsystem for Linux** → restart
  4. Run `wsl --update` in an Administrator terminal afterward

### 6.3 `failed to connect to the docker API at npipe:////./pipe/docker\_engine`

* **Cause:** Docker Desktop application isn't actually running yet — the CLI exists but the engine backend isn't started.
* **Solution:** Launch Docker Desktop from the Start menu, wait for the whale icon in the system tray to go steady (not spinning), then retry.

### 6.4 `Server: ERROR: request returned 500 Internal Server Error ... dockerDesktopLinuxEngine`

* **Cause:** WSL2 backend failed to initialize properly (often right after fixing the virtualization issue, before a full restart cycle completed).
* **Solution (in order of escalation):**

  1. Fully quit Docker Desktop (tray icon → Quit) and reopen it
  2. `wsl --shutdown` (as Administrator) then reopen Docker Desktop
  3. `wsl --update` (as Administrator), then restart the PC
  4. Docker Desktop → Settings → General → confirm "Use the WSL 2 based engine" is checked
  5. Last resort: Docker Desktop → Settings → Troubleshoot → "Reset to factory defaults"

### 6.5 `ports are not available: exposing port TCP 0.0.0.0:3306 ... bind: Only one usage of each socket address is normally permitted`

* **Cause:** Port 3306 on the host machine was already occupied — by the original XAMPP MySQL service still running in the background.
* **Solution:** Remap the **host-side** port only in `docker-compose.yml` (left number in `"3316:3306"`), leaving the container-internal port at `3306`. XAMPP and Docker can then coexist without conflict.

### 6.6 `ERROR 1286 (42000): Unknown storage engine 'FEDERATED'`

* **Cause:** In MariaDB, this engine is actually called **FederatedX**, and it is *not* auto-enabled just by adding `federated` to a `.cnf` config file the way one might expect. It must be explicitly loaded as a plugin.
* **Solution:** Connect as root and run:

```sql
  INSTALL SONAME 'ha\_federatedx';
  ```

  Then confirm with `SHOW ENGINES;` — look for `FEDERATED` with `Support: YES`. Must be repeated on **both** nodes if both will ever host a FEDERATED link table. This needs to be re-run only if the container's data volume is wiped (e.g. `docker-compose down -v`).

### 6.7 `ERROR 1064 (42000): You have an error in your SQL syntax ... near 'docker exec -it pc2...'`

* **Cause:** Typed a `docker exec` (Windows/Docker) command while still inside the MariaDB SQL prompt. MariaDB tried to interpret it as SQL.
* **Solution:** Always check which prompt is active before typing:

  * `MariaDB \[company\_db]>` → SQL only, statements end in `;`
  * `C:\\Users\\...>` → Windows Command Prompt, `docker ...` commands only
Type `exit;` inside MariaDB to return to Command Prompt before running any `docker` command.

### 6.8 MariaDB prompt "stuck" waiting for more input (shows `->` repeatedly)

* **Cause:** A statement was started but never terminated with `;`, so MariaDB keeps waiting for the rest of it (this happens if a `docker exec ...` line gets typed into the SQL prompt by mistake).
* **Solution:** Type `\\c` and press Enter to clear the pending statement and reset the prompt cleanly.

### 6.9 `'SELECT' is not recognized as an internal or external command`

* **Cause:** The reverse of 6.7 — typing a SQL command into regular Windows Command Prompt instead of inside MariaDB. Often happens because a `docker exec -it ... mysql ...` session (without `-e "..."`) closes back to Command Prompt right after its one command finishes.
* **Solution:** Reconnect with `docker exec -it pc1 mysql -u fed\_user -p<db\_password> company\_db` and confirm the prompt shows `MariaDB \[company\_db]>` before typing SQL. Alternatively, use the one-shot form for single queries:

```
  docker exec -it pc2 mysql -u fed\_user -p<db\_password> company\_db -e "SELECT \* FROM customers\_ctg;"
  ```

  which runs, prints, and exits automatically — no manual `exit` needed after.

### 6.10 `Warning: World-writable config file '/etc/mysql/conf.d/federated.cnf' is ignored`

* **Cause:** Windows file permissions on the mounted `.cnf` file are more permissive than Linux expects (cosmetic issue from bind-mounting a Windows file into a Linux container).
* **Solution:** Harmless — can be ignored. Since the FEDERATED engine is actually enabled via `INSTALL SONAME` (SQL command) rather than this config file, the warning has no functional impact.

\---

## 7\. Key Command Reference (Cheat Sheet)

```
# Container lifecycle
docker-compose up -d          # start both nodes
docker-compose stop           # stop, keep data
docker-compose start          # resume
docker-compose down -v        # full reset — wipes data, plugin installs, everything
docker ps                     # check running containers
docker-compose logs pc1       # view logs if a container misbehaves

# Connecting
docker exec -it pc1 mysql -u root -p<root\_password>                     # root session
docker exec -it pc1 mysql -u fed\_user -p<db\_password> company\_db   # app-user session
docker exec -it pc2 mysql -u fed\_user -p<db\_password> company\_db -e "SELECT ...;"  # one-shot query

# Inside MariaDB
\\c        # clear a stuck/incomplete statement
exit;     # leave MariaDB, return to Windows Command Prompt
```

\---

## 8\. Limitations of This Setup (worth noting in any write-up/viva)

* **No fault tolerance** — if pc2 goes down, queries touching `customers\_ctg\_link` fail outright; there's no failover or replica to fall back on.
* **No distributed transactions** — a multi-node write isn't atomic; if a failure happens mid-operation, there's no two-phase commit to roll everything back consistently.
* **Performance** — every FEDERATED query round-trips over the network to the remote node; no query optimization pushes filters down efficiently in all cases.
* **FederatedX itself is a deprecated/legacy engine** in current MariaDB — fine for learning fragmentation concepts, but MariaDB's own docs point to the newer **SPIDER** or **CONNECT** engines for anything production-oriented.

\---

## 9\. Summary

This lab successfully demonstrates the core distributed database concept of **horizontal fragmentation with transparency**: two independently running MariaDB nodes (in Docker containers) each hold a different slice of a logical `customers` table, linked via the FederatedX storage engine, and unified into a single queryable view (`customers\_global`). A client querying that view cannot tell — and doesn't need to know — that half the data lives on a completely separate server.

