# Evidence Pack - Week 3: Deployment & Evidence

**Group:** 7
**Members:** Bùi Thành Nghĩa, Lê Thị Thùy Trang, Trần Minh Quang, Hoàng Kim Hùng, Nguyễn Công Thịnh, Phạm Công Huy, Nguyễn Tất Văn, Lê Nguyễn Nhật Thành, Đỗ Phúc
**Project:** Sports Field Booking System (AWS 3-Tier Architecture)

---

## 1. Project Foundation
* **Chosen Database Engine & Paradigm:** Amazon RDS PostgreSQL / Relational
* **W2 Evidence Link:** [Insert your team's W2 markdown or GitHub commit link here]

---

## 2. Data Access Pattern Log

---

## Part A — Three Core Data Access Patterns

### Pattern 1 — Booking Creation & Payment Processing
Create multiple `datsans` records (one per timeslot), generate PayOS payment link, then atomically update `tinh_trang = true` on successful payment confirmation  
→ **~100–150 calls/min during peak hours (6 PM–9 PM, weekends)**

---

### Pattern 2 — User Booking History Timeline
SELECT `datsans` joined with `vitrisans`, `santhethaos`, `danhgias`, filtered by `id_nguoi_dung`, ordered by `ngay_dat DESC` with facility name and rating  
→ **~20–30 calls/min (steady; triggered when user opens booking history page)**

---

### Pattern 3 — Revenue & Facility Analytics (Aggregation)
GROUP BY facility (`santhethaos.ten_san`) and `SUM(thanh_tien)` or `COUNT(datsans.id)` where `id_chu_san = :owner_id`, spanning joins:  
`datsans → vitrisans → santhethaos`  
→ **~5–10 calls/day (business intelligence dashboard, daily report generation)**

---

## Part B — Paradigm Analysis & Index Strategy

### Pattern 1 — Relational + B-tree Composite Index

**Paradigm Effectiveness**  
PostgreSQL's relational model enforces referential integrity across  
(`datsans → vitrisans → santhethaos`) via foreign keys, ensuring bookings cannot reference non-existent facilities.

Transactions guarantee all-or-nothing atomicity:  
multiple `INSERT datsans` + `UPDATE tinh_trang` complete together or both fail (no partial bookings).

**Index**
- Composite B-tree on `(orderCode, tinh_trang)`  
  → locates pending payment records in **O(log n)** without table scan  
- Simple B-tree on `(id_nguoi_dung)`  
  → retrieves user's bookings by ROWID lookup

**Self-Hosted EC2 Trade-off**
- Eliminates **$200–500/month AWS RDS baseline cost + per-GB backup charges**
- Sacrifice automated failover:
  - **RPO ~1 hour** via `pg_basebackup` to secondary EC2 + streaming replication  
  - **RTO ~5 min** manual promotion  

**Chosen Strategy**
- Budget optimization
- Production recommendation:
  - Cross-AZ EBS snapshots hourly
  - Automated failover (Patroni or repmgr)

**Estimated Cost**
- ~$50/month compute vs. ~$300/month RDS redundancy

---

### Pattern 2 — Relational + Clustered Foreign-Key Indexes

**Paradigm Effectiveness**  
Multi-table JOINs (5 tables) are native to relational design.

Filter push-down on:
- `(id_nguoi_dung, tinh_trang = true)`

→ reduces intermediate result sets:
- From all bookings → single user's 10–20 bookings before joining

Non-relational (document) stores lack built-in foreign-key enforcement, requiring application-layer validation.

**Index**
- B-tree on `datsans(id_nguoi_dung)`  
- B-tree on `danhgias(id_vi_tri_san)`  
  → enables nested-loop join with early termination  

- B-tree on `vitrisans(id_san)`  
  → enables single-hop lookup to `santhethaos`

**Self-Hosted Strategy**
- WAL archival to S3 (`pg_basebackup + s3:// URI on secondary`)

**Recovery Metrics**
- **RTO:** 30 min (restore from snapshot + replay WAL)  
- **RPO:** ~15 min  

**Cost**
- 1 large EC2 (m5.xlarge ~$150/mo)
- micro standby ($30/mo)

→ vs. RDS Multi-AZ ($450/mo)

**Trade-off**
- Lose real-time HA  
- Gain control over upgrade windows

---

### Pattern 3 — Relational + Aggregate-Aware Index

**Paradigm Effectiveness**  
GROUP BY + SUM/COUNT aggregations are relational specialties.

PostgreSQL planner uses:
- **Index-Only Scans (IOS)** when `(id_chu_san, thanh_tien)` are indexed  
→ avoids heap lookups entirely (reads only index pages)

Document stores (MongoDB):
- Must scan full documents for aggregation  
→ ~3–5× slower without pre-computed counters

**Index**
- Composite B-tree `(id_chu_san, thanh_tien)` → enables IOS  
- Partial index on `tinh_trang = true`  
  → excludes cancelled orders from scan (avoid dead tuples in MVCC)

**Self-Hosted Rationale**
- Analytics jobs:
  - 10–30 sec scans of 10M rows  
  - batch-scheduled nightly  
  - no real-time failover SLA  

**Backup Strategy**
- Single EC2
- Hourly EBS snapshots
- Weekly full backups to S3 Glacier

**Cost**
- Storage: ~$5/mo  
- Restore: ~$1 per restore  

**Recovery**
- **RTO ~2 hours** acceptable

**Conclusion**
- RDS would cost ~$300/mo for identical performance

---

## Part C — Wrong-Paradigm Test (Pattern 1: Booking Creation)

**Why Key-Value Store (Redis/Memcached) Fails**

A pure Key-Value store (e.g., Redis, Memcached) cannot atomically enforce the constraint:

- `"orderCode must be globally unique"`
- `"tinh_trang must only transition false → true"`

Redis transactions (`WATCH/MULTI/EXEC`):
- Lack isolation levels  
- Cannot serialize:
  1. INSERT datsans #1  
  2. INSERT datsans #2  
  3. UPDATE `tinh_trang = true`

**Failure Scenario**
- Two concurrent requests generate same `orderCode`
- No foreign-key or UNIQUE constraint → data corruption

**Workaround (but costly)**
- Distributed locking (Redlock):
  - Requires 3-node setup (~$150/mo)
  - Adds **5–30ms latency per booking**
  - At 100 req/min → 100+ lock ops/min → contention risk

**Relational Advantage**
- PostgreSQL:
  - UNIQUE constraint  
  - ACID transactions  
  - SERIALIZABLE isolation  

→ Detects conflict **atomically in microseconds**

---

---

## 3. Deployment Evidence

### AC #1: Database Instance Setup & Network Isolation
* **Evidence:** > *[Insert AWS Console Screenshot here: RDS configuration showing Engine = PostgreSQL, Status = Available, and the VPC Subnet Group clearly indicating placement in Private Subnets]*
* **Technical Notes:** The RDS instance is deployed within a Private Subnet group. We explicitly disabled "Public Access." This architectural decision ensures the database is completely isolated from the public internet, mitigating direct external attack vectors. Only the Application Tier (EC2 instances) can route traffic to it.

### AC #2: Data Encryption at Rest
* **Evidence:** > *[Insert AWS Console Screenshot here: RDS Storage configuration showing Encryption = Enabled and referencing the KMS Key alias]*
* **Technical Notes:** Storage encryption at rest is enforced using the AWS-managed KMS key (`aws/rds`). We chose the managed key approach over a Customer Managed Key (CMK) to minimize operational overhead regarding key rotation while still satisfying strict data protection compliance for user PII and payment records.

---

## 4. Working Query Evidence

### Query 1: Relational JOIN (User Booking History)
* **Description:** A multi-table `JOIN` operation to construct a comprehensive view of a user's past bookings, resolving foreign keys across `datsans`, `vitrisans`, `santhethaos`, and `danhgias`.
* **Evidence:**
    > *[Insert Terminal/DBeaver Screenshot here: Showing the SQL query execution and the actual returned rows]*
* **Operation:**
    ```sql
    SELECT danhgias.id AS id_danh_gia,
           datsans.id_vi_tri_dat_san,
           datsans."orderCode",
           datsans.ngay_dat,
           datsans.gio_dat,
           datsans.thanh_tien,
           santhethaos.ten_san,
           santhethaos.huyen,
           vitrisans.so_san
    FROM datsans
    JOIN vitrisans ON datsans.id_vi_tri_dat_san = vitrisans.id
    JOIN santhethaos ON santhethaos.id = vitrisans.id_san
    LEFT JOIN danhgias ON vitrisans.id = danhgias.id_vi_tri_san
    WHERE datsans.id_nguoi_dung = 'user-uuid-1234';
    ```

### Query 2: Indexed Lookup (Authentication)
* **Description:** Retrieving user credentials via email lookup. To prevent a Full Table Scan (Sequential Scan), an index is utilized to ensure $O(\log n)$ lookup time.
* **Evidence:**
    > *[Insert Terminal/DBeaver Screenshot here: Showing the EXPLAIN ANALYZE output proving that an Index Scan was used instead of a Seq Scan]*
* **Operation:**
    ```sql
    EXPLAIN ANALYZE
    SELECT *
    FROM nguoidungs
    WHERE email = 'user@example.com';
    ```

---

## 5. Lambda + Bedrock Evidence

### AC #1: Function Invocation & Logs
* **Evidence:** > *[Insert CloudWatch Logs Screenshot here: Showing the log stream timestamped immediately after the Lambda trigger event]*
* **Technical Notes:** The Lambda function is triggered via an API Gateway integration. The IAM Execution Role adheres to the **Principle of Least Privilege**, with specific `Allow` actions scoped strictly to the required Bedrock Knowledge Base ARN, containing absolutely no `Resource: "*"` or `Action: "*"` statements.

### AC #2: AI Retrieval Execution
* **Evidence:** > *[Insert Terminal Screenshot here: Showing the CLI payload or application response containing the actual Retrieve/RetrieveAndGenerate output]*
* **CLI Command Executed:**
    ```bash
    aws lambda invoke --function-name sports-booking-ai-helper --payload '{"query": "How to cancel a booking?"}' response.json
    ```

---

## 6. VPC & Networking Evidence

### AC #1: S3 Gateway Endpoint
* **Evidence:** > *[Insert VPC Route Table Screenshot here: Showing the Private Subnet route table with a target pointing to a `vpce-xxxx` (Gateway Endpoint) for the `com.amazonaws.region.s3` destination]*
* **Technical Notes:** Implementing a Gateway Endpoint ensures that traffic between our private EC2 instances and S3 never traverses the public internet, significantly enhancing security and eliminating NAT Gateway data processing charges for internal traffic.

### AC #2: Database Security Group Rules
* **Evidence:** > *[Insert Security Group Screenshot here: Showing the Inbound Rules for the RDS SG]*
* **Technical Notes:** The inbound rule explicitly references the Security Group ID of the Application Tier (`sg-0abcd1234...`) on port 5432, rather than a CIDR block. This creates a logical security boundary: even if a new EC2 instance is launched in the private subnet, it cannot access the database unless it is explicitly attached to the Application Security Group.

---

## 7. Negative Security Test

* **Scenario:** Attempting to establish a direct connection to the RDS endpoint from an external, unauthorized environment (a local developer machine outside the AWS VPC).
* **Expected Result:** The connection must fail (timeout) because the database resides in a private subnet with no Internet Gateway route, and the Security Group drops uninvited packets.
* **Evidence:**
    > *[Insert Terminal Screenshot here: Showing the `psql -h <rds-endpoint> -U postgres` command timing out]*

---

## 8. Bonus: Real-world Ops Scenario (Multi-AZ Failover Drill)

* **Action Taken:** Simulated a severe hardware degradation by triggering a "Reboot with failover" via the RDS Console on our Multi-AZ deployment.
* **Pre-execution State:** Primary DB located in `ap-southeast-1a`. Application functioning normally.
    > *[Insert Screenshot: RDS Console showing current AZ]*
* **Post-execution State:** Standby DB in `ap-southeast-1b` was automatically promoted to Primary. CNAME record propagated.
    > *[Insert Screenshot: RDS Console showing the new AZ and "Available" status]*
* **Measurement:** Connection drop observed for exactly **[Insert Time, e.g., 85 seconds]** before the application successfully re-established the connection pool.
* **Architectural Reflection:** While Multi-AZ doubles compute and storage costs, the drill proved its value for critical workloads. The failover is handled entirely by AWS at the DNS level. A key takeaway is the importance of configuring our Node.js/Sequelize connection pool to aggressively retry on connection loss, rather than crashing the application entirely.
