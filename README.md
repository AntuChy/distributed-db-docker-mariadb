
# Distributed Database Environment using Docker + MariaDB (FEDERATED Engine)

**Author:** Antu Chowdhury

**Topic:** Building a Simple Distributed Database using Data Fragmentation

**Tools Used:** Docker Desktop, MariaDB 10.11 (via Docker), FEDERATED/FederatedX Storage Engine

> **Note:** All passwords in this report (`<root_password>`, `<db_password>`) are placeholders. Replace them with your own values before running any command — do not use literal angle-bracket text as a real password.

---

## 1. Objective

To design and implement a simple **distributed database environment** demonstrating **horizontal data fragmentation** across two independent database nodes, queried transparently as a single logical database — without the client needing to know where each piece of data physically resides.

---

## 2. Core Concept: What "Distributed Database" Means Here

A distributed database is not simply "two databases that talk to each other" — the defining property is **transparency**. A user or application should be able to run one query and get correct results, regardless of which physical node actually stores each row.

There are two common ways people implement something they *call* "distributed":

|Approach|What it means|What we built|
|-|-|-|
|**Replication**|Every node has a *full copy* of the same data (used for redundancy/availability)|❌ Not this|
|**Fragmentation**|Data is *split* across nodes — each node holds a different slice|✅ This|

We implemented **horizontal fragmentation**: rows of a `customers` table were split by region — Dhaka customers physically live on Node 1 (`pc1`), Chittagong customers physically live on Node 2 (`pc2`) — then unified into one queryable view.

### Key DBMS terms demonstrated

* **Fragmentation** — splitting a table's rows (horizontal) or columns (vertical) across nodes.
* **Transparency** — the client queries `customers_global` and doesn't need to know 2 of the rows come from a completely different server.
* **FEDERATED / FederatedX Engine** — a MariaDB storage engine that creates a *local proxy table* pointing at a table on a remote server. Reading/writing to the proxy table transparently forwards the operation to the remote server via MySQL's client protocol (`libmysql`).

---

## 3. Architecture

```text
                 ┌──────────────────────────────────────────────┐
                 │          Docker Network: distdb-net          │
                 │               (Bridge Network)               │
                 └──────────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
      ┌───────▼────────┐                ┌───────▼────────┐
      │  PC1 (MariaDB) │                │  PC2 (MariaDB) │
      │  Port: 3316    │                │  Port: 3307    │
      │ (3306 inside)  │                │ (3306 inside)  │
      └────────────────┘                └────────────────┘
              │                                 │
              │                                 │
      ┌──────────────────┐              ┌──────────────────┐
      │ customers_dhaka  │              │ customers_ctg    │
      │   (Real Table)   │              │   (Real Table)   │
      └──────────────────┘              └──────────────────┘
              │
              │
      ┌─────────────────────┐
      │ customers_ctg_link  │──────────────────────────────►
      │  (FEDERATED Table)  │   Connects to customers_ctg
      └─────────────────────┘        on PC2
              │
              ▼
      ┌─────────────────────┐
      │ customers_global    │
      │       (VIEW)        │
      ├─────────────────────┤
      │ customers_dhaka     │
      │      UNION ALL      │
      │ customers_ctg_link  │
      └─────────────────────┘

```
We chose **Docker containers** instead of two physical PCs on a LAN because Docker's internal DNS (container name resolution: `pc1`, `pc2`) eliminates the need for IP address management, port forwarding across a real router, and Windows Firewall configuration — all of which caused significant friction when first attempting this on real hardware over Wi-Fi/LAN.

---

## 4. Full Setup — Step by Step

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
    container_name: pc1
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: company_db
      MYSQL_USER: fed_user
      MYSQL_PASSWORD: <db_password>
    ports:
      - "3316:3306"
    volumes:
      - ./pc1-config/federated.cnf:/etc/mysql/conf.d/federated.cnf
    networks:
      - distdb-net

  pc2:
    image: mariadb:10.11
    container_name: pc2
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: company_db
      MYSQL_USER: fed_user
      MYSQL_PASSWORD: <db_password>
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

> **Note:** Host ports (left side of `3316:3306`) had to be changed from the default `3306` because a local XAMPP MySQL install was already using it on the host machine. The **internal** port (right side, `3306`) stays standard — that's what containers use to reach each other, and it's what matters for the FEDERATED connection string.

### 4.4 Launch containers

```
docker-compose up -d
docker ps          # confirm both pc1 and pc2 show "Up"
```

### 4.5 Enable the FEDERATED engine (must be done via SQL, not just the config file)

On **each** container:

```
docker exec -it pc1 mysql -u root -p<root_password>
```

```sql
INSTALL SONAME 'ha_federatedx';
SHOW ENGINES;   -- confirm FEDERATED shows Support: YES
```

Repeat for `pc2`.

### 4.6 Create the data fragments

**On pc1** (`docker exec -it pc1 mysql -u fed_user -p<db_password> company_db`):

```sql
CREATE TABLE customers_dhaka (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
);
INSERT INTO customers_dhaka VALUES
(1, 'Rafiq Islam', 'Dhaka', 'rafiq@example.com'),
(2, 'Nusrat Jahan', 'Dhaka', 'nusrat@example.com');
```

**On pc2** (`docker exec -it pc2 mysql -u fed_user -p<db_password> company_db`):

```sql
CREATE TABLE customers_ctg (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
);
INSERT INTO customers_ctg VALUES
(3, 'Farhan Kabir', 'Chittagong', 'farhan@example.com'),
(4, 'Sabrina Akter', 'Chittagong', 'sabrina@example.com');
```

### 4.7 Create the FEDERATED link (on pc1, pointing at pc2)

```sql
CREATE TABLE customers_ctg_link (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  region VARCHAR(50),
  email VARCHAR(100)
) ENGINE=FEDERATED
CONNECTION='mysql://fed_user:<db_password>@pc2:3306/company_db/customers_ctg';
```

Note the connection string uses the **container name** (`pc2`) and **internal port** (`3306`) — not the host-mapped port.

### 4.8 Build the unified view (on pc1)

```sql
CREATE VIEW customers_global AS
SELECT * FROM customers_dhaka
UNION ALL
SELECT * FROM customers_ctg_link;
```

### 4.9 Verify

```sql
SELECT * FROM customers_global;
```

Expected: all 4 rows — 2 stored locally on pc1, 2 fetched live from pc2 in real time.

---

## 5. Orders Fragment — Full Setup \& Cross-Node JOIN Demo

This second fragment is what elevates the demo from a simple `UNION` of two tables into a genuine test of distributed querying: a **JOIN across two physically separate nodes**. `customers` data and `orders` data both stay fragmented (Dhaka vs Chittagong), and we prove a query engine can still relate them correctly as if they lived in one database.

### 5.1 Create the orders fragment on pc1 (Dhaka orders)

```
docker exec -it pc1 mysql -u fed_user -p<db_password> company_db
```

```sql
CREATE TABLE orders_dhaka (
  order_id INT PRIMARY KEY,
  customer_id INT,
  amount DECIMAL(10,2)
);

INSERT INTO orders_dhaka VALUES
(101, 1, 1500.00),
(102, 2, 2300.00);
```

(`customer_id` 1 and 2 refer to Rafiq and Nusrat, who live in `customers_dhaka`.)

### 5.2 Create the orders fragment on pc2 (Chittagong orders)

```
docker exec -it pc2 mysql -u fed_user -p<db_password> company_db
```

```sql
CREATE TABLE orders_ctg (
  order_id INT PRIMARY KEY,
  customer_id INT,
  amount DECIMAL(10,2)
);

INSERT INTO orders_ctg VALUES
(201, 3, 1800.00),
(202, 4, 950.00);
```

(`customer_id` 3 and 4 refer to Farhan and Sabrina, who live in `customers_ctg`.)

### 5.3 Link pc2's orders into pc1 via FEDERATED

Back on pc1:

```sql
CREATE TABLE orders_ctg_link (
  order_id INT PRIMARY KEY,
  customer_id INT,
  amount DECIMAL(10,2)
) ENGINE=FEDERATED
CONNECTION='mysql://fed_user:<db_password>@pc2:3306/company_db/orders_ctg';
```

Test the link on its own first:

```sql
SELECT * FROM orders_ctg_link;
```

Expected: the two Chittagong orders (201, 202), fetched live from pc2.

### 5.4 Build the unified orders view

Still on pc1:

```sql
CREATE VIEW orders_global AS
SELECT * FROM orders_dhaka
UNION ALL
SELECT * FROM orders_ctg_link;
```

Test it:

```sql
SELECT * FROM orders_global;
```

Expected: all 4 orders (101, 102, 201, 202) — 2 physically local, 2 pulled from pc2.

### 5.5 The cross-node JOIN — customers + orders together

This is the real test: joining `customers_global` (itself already spanning 2 nodes) with `orders_global` (also spanning 2 nodes):

```sql
SELECT c.name, c.region, o.order_id, o.amount
FROM customers_global c
JOIN orders_global o ON c.id = o.customer_id
ORDER BY c.id;
```

Expected output:

```
+---------------+-------------+----------+---------+
| name          | region      | order_id | amount  |
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
SELECT region, SUM(amount) AS total_sales
FROM customers_global c
JOIN orders_global o ON c.id = o.customer_id
GROUP BY region;
```

Expected output:

```
+-------------+-------------+
| region      | total_sales |
+-------------+-------------+
| Dhaka       | 3800.00     |
| Chittagong  | 2750.00     |
+-------------+-------------+
```

This shows region-level totals computed from data that was never physically stored together — the aggregation happens transparently across nodes.

### 5.7 Write operations through the FEDERATED link (proving it's bidirectional)

```sql
-- INSERT: lands on pc2, not pc1
INSERT INTO orders_ctg_link VALUES (203, 4, 500.00);

-- UPDATE: modifies the row on pc2
UPDATE orders_ctg_link SET amount = 600.00 WHERE order_id = 203;

-- DELETE: removes the row from pc2
DELETE FROM orders_ctg_link WHERE order_id = 203;
```

Verify any of these directly on pc2 to confirm the write actually happened remotely:

```
docker exec -it pc2 mysql -u fed_user -p<db_password> company_db -e "SELECT * FROM orders_ctg;"
```

### 5.8 Fault-tolerance check (demonstrating a real limitation)

```
docker stop pc2
```

Then on pc1:

```sql
SELECT * FROM orders_global;
```

This will **error out** on the `orders_ctg_link` portion — proving FEDERATED has no replication or failover; if the remote node is down, that fragment is simply unavailable. Restart pc2 afterward:

```
docker start pc2
```

### 5.9 Query plan inspection

```sql
EXPLAIN SELECT c.name, o.amount
FROM customers_global c
JOIN orders_global o ON c.id = o.customer_id;
```

Shows how the optimizer treats the FEDERATED-backed portions differently from purely local tables — useful talking point on distributed query cost.

---

## 6. Errors Encountered & Solutions (Full Troubleshooting Log)

### 6.1 `Virtualization support not detected` (Docker Desktop startup error)

- **Cause:** Hardware virtualization (Intel VT-x) disabled in BIOS.
- **Solution:**
  1. Task Manager → **Performance** → **CPU** → check whether **Virtualization** is Enabled.
  2. If Disabled: Restart → enter BIOS (Lenovo: **F1** or **F2**) → **Security** → **Intel Virtualization Technology (VT-x)** → **Enabled** → Save & Exit (**F10**).
  3. Enable Windows features: **Virtual Machine Platform** and **Windows Subsystem for Linux**.
  4. Run `wsl --update` in an Administrator terminal.

### 6.2 `failed to connect to the Docker API at npipe:////./pipe/docker_engine`

- **Cause:** Docker Desktop was not running.
- **Solution:** Launch Docker Desktop, wait until it finishes starting, and then retry the command.

### 6.3 `Server: ERROR: request returned 500 Internal Server Error ... dockerDesktopLinuxEngine`

- **Cause:** WSL2 backend failed to initialize correctly.
- **Solution:**
  1. Restart Docker Desktop.
  2. Run `wsl --shutdown`.
  3. Run `wsl --update` and restart the PC.
  4. Ensure **Use the WSL 2 based engine** is enabled.
  5. If necessary, reset Docker Desktop to factory defaults.

### 6.4 `Ports are not available ... bind: Only one usage of each socket address is normally permitted`

- **Cause:** Host port **3306** was already in use by XAMPP MySQL.
- **Solution:** Change the host port mapping in `docker-compose.yml` (e.g., `3316:3306`) while keeping the container port as `3306`.

### 6.5 `ERROR 1286 (42000): Unknown storage engine 'FEDERATED'`

- **Cause:** MariaDB uses the **FederatedX** plugin, which must be loaded manually.
- **Solution:**

```sql
INSTALL SONAME 'ha_federatedx';
```

Then verify using:

```sql
SHOW ENGINES;
```

Repeat on each database node if needed.

### 6.6 `ERROR 1064 (42000): You have an error in your SQL syntax ... near 'docker exec -it ...'`

- **Cause:** A Docker command was entered inside the MariaDB SQL prompt.
- **Solution:** Run Docker commands only from the terminal. Type `exit;` to leave the MariaDB prompt before executing Docker commands.

### 6.7 MariaDB prompt continuously shows `->`

- **Cause:** An SQL statement was started but not terminated with `;`.
- **Solution:** Type:

```sql
\c
```

to cancel the current statement.

### 6.8 `'SELECT' is not recognized as an internal or external command`

- **Cause:** An SQL command was executed from the Windows Command Prompt instead of the MariaDB shell.
- **Solution:** Reconnect to MariaDB using:

```bash
docker exec -it pc1 mysql -u fed_user -p company_db
```

or execute a single query using:

```bash
docker exec -it pc2 mysql -u fed_user -p company_db -e "SELECT * FROM customers_ctg;"
```

### 6.9 `Warning: World-writable config file '/etc/mysql/conf.d/federated.cnf' is ignored`

- **Cause:** Windows file permissions on the mounted configuration file are more permissive than Linux expects.
- **Solution:** This warning is harmless and can be ignored because the FEDERATED engine is enabled using `INSTALL SONAME`, not through the configuration file.
## 7. Key Command Reference (Cheat Sheet)

```
# Container lifecycle
docker-compose up -d          # start both nodes
docker-compose stop           # stop, keep data
docker-compose start          # resume
docker-compose down -v        # full reset — wipes data, plugin installs, everything
docker ps                     # check running containers
docker-compose logs pc1       # view logs if a container misbehaves

# Connecting
docker exec -it pc1 mysql -u root -p<root_password>                     # root session
docker exec -it pc1 mysql -u fed_user -p<db_password> company_db   # app-user session
docker exec -it pc2 mysql -u fed_user -p<db_password> company_db -e "SELECT ...;"  # one-shot query

# Inside MariaDB
\\c        # clear a stuck/incomplete statement
exit;     # leave MariaDB, return to Windows Command Prompt
```

---

## 8. Limitations of This Setup (worth noting in any write-up/viva)

* **No fault tolerance** — if pc2 goes down, queries touching `customers_ctg_link` fail outright; there's no failover or replica to fall back on.
* **No distributed transactions** — a multi-node write isn't atomic; if a failure happens mid-operation, there's no two-phase commit to roll everything back consistently.
* **Performance** — every FEDERATED query round-trips over the network to the remote node; no query optimization pushes filters down efficiently in all cases.
* **FederatedX itself is a deprecated/legacy engine** in current MariaDB — fine for learning fragmentation concepts, but MariaDB's own docs point to the newer **SPIDER** or **CONNECT** engines for anything production-oriented.

---

## 9. Summary

This lab successfully demonstrates the core distributed database concept of **horizontal fragmentation with transparency**: two independently running MariaDB nodes (in Docker containers) each hold a different slice of a logical `customers` table, linked via the FederatedX storage engine, and unified into a single queryable view (`customers_global`). A client querying that view cannot tell — and doesn't need to know — that half the data lives on a completely separate server.
