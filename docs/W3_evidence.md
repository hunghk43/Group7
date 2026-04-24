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

### Part A: Access Patterns (Real-world App Scenarios)
| ID | User Functionality | Frequency | Priority |
| :--- | :--- | :--- | :--- |
| **1** | Search fields by district/city and field type | High | Must-have |
| **2** | Create a booking (transactional: select field, date, and time slot) | High | Must-have |
| **3** | View user booking history | Medium | Must-have |
| **4** | Staff/Owner views booking history by field | Medium | Must-have |
| **5** | User authentication (lookup by email) | High | Must-have |

### Part B: Engine, Paradigm & Reasoning
* **Engine:** Amazon RDS PostgreSQL
* **Paradigm:** Relational
* **Architectural Reasoning:** The sports field booking domain is inherently relational. Entities like `Users`, `Fields`, `Bookings`, and `Reviews` have strict dependencies (Foreign Keys). Core business operations, such as generating booking histories or staff reports, require multi-table `JOIN` aggregations. Furthermore, creating a booking involves financial and availability state changes that demand strict ACID compliance to prevent double-booking.
* **HA & Backup Plan (Managed Service Strategy):** We leverage RDS automated backups with a 7-day retention period. For production, Multi-AZ deployment would be enabled for synchronous replication and automatic failover.
* **Cost Estimate:** ~$30-40/month for a `db.t3.micro` Single-AZ instance (Dev/Test environment) including allocated GP3 storage.

### Part C: The "Wrong-Paradigm" Test
If we were to use a **Key-Value database (like DynamoDB)** for this application, resolving multi-table relationships (e.g., fetching a booking along with user details, field location, and payment status) would require either executing multiple sequential queries from the application layer or duplicating massive amounts of data across items to avoid `JOIN`s. Complex ad-hoc filtering (like searching for available fields by multiple criteria) would necessitate costly `Scan` operations or managing an unmaintainable number of Global Secondary Indexes (GSIs), ultimately degrading performance and increasing operational overhead.

---

## 3. Deployment Evidence

### AC #1: Database Instance Setup & Network Isolation
* **Evidence:** 
    ![RDS Configuration](../../pics/RDS%20config.png)
    ![RDS Overview](../../pics/RDS%20ovr.png)
* **Technical Notes:** The RDS instance is deployed within a Private Subnet group. We explicitly disabled "Public Access." This architectural decision ensures the database is completely isolated from the public internet, mitigating direct external attack vectors. Only the Application Tier (EC2 instances) can route traffic to it.

### AC #2: Data Encryption at Rest
* **Evidence:** 
    ![RDS Security Configuration](../../pics/RDS%20sec.png)
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
* **Evidence:** 
    ![S3 Endpoint Configuration](../../pics/s3%20endpoint.png)
    ![VPC Architecture](../../pics/vpc.png)
* **Technical Notes:** Implementing a Gateway Endpoint ensures that traffic between our private EC2 instances and S3 never traverses the public internet, significantly enhancing security and eliminating NAT Gateway data processing charges for internal traffic.

### AC #2: Database Security Group Rules
* **Evidence:** 
    ![RDS Inbound Security Group Rules](../../pics/RDS%20inbound%20sg.png)
    ![RDS Outbound Security Group Rules](../../pics/RDS%20outbound.png)
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
