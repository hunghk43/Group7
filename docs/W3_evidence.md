# Evidence Pack - Week 3: Deployment & Evidence

**Group:** 7
**Members:** Bùi Thành Nghĩa, Lê Thị Thùy Trang, Trần Minh Quang, Hoàng Kim Hùng, Nguyễn Công Thịnh, Phạm Công Huy, Nguyễn Tất Văn, Lê Nguyễn Nhật Thành, Đỗ Phúc
**Project:** Sports Field Booking System (AWS 3-Tier Architecture)

---

## 1. Project Foundation
* **Chosen Database Engine & Paradigm:** Amazon RDS PostgreSQL / Relational
* **W2 Evidence Link:** 
![Diagram tuần 2](XB-WEEK-2.png)
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
    ![RDS Configuration](RDS%20config.png)
    ![RDS Overview](RDS%20ovr.png)
* **Technical Notes:** The RDS instance is deployed within a Private Subnet group. We explicitly disabled "Public Access." This architectural decision ensures the database is completely isolated from the public internet, mitigating direct external attack vectors. Only the Application Tier (EC2 instances) can route traffic to it.

### AC #2: Data Encryption at Rest
* **Evidence:** 
    ![RDS Security Configuration](RDS%20sec.png)
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


---

## 6. VPC & Networking Evidence

### AC #1: S3 Gateway Endpoint
* **Evidence:** 
    ![S3 Endpoint Configuration](s3%20endpoint.png)
    ![VPC Architecture](vpc.png)
* **Technical Notes:** Implementing a Gateway Endpoint ensures that traffic between our private EC2 instances and S3 never traverses the public internet, significantly enhancing security and eliminating NAT Gateway data processing charges for internal traffic.

### AC #2: Database Security Group Rules
* **Evidence:** 
    ![RDS Inbound Security Group Rules](RDS%20inbound%20sg.png)
    ![RDS Outbound Security Group Rules](RDS%20outbound.png)
* **Technical Notes:** The inbound rule explicitly references the Security Group ID of the Application Tier (`sg-0abcd1234...`) on port 5432, rather than a CIDR block. This creates a logical security boundary: even if a new EC2 instance is launched in the private subnet, it cannot access the database unless it is explicitly attached to the Application Security Group.




