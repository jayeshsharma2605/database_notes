# DynamoDB for System Design Interviews

> Distributed Systems Study Library · Author: Jayesh Sharma

Beginner-friendly notes with simple examples, memory tricks, diagrams, and interview revision.

## 1. What is DynamoDB?

Amazon DynamoDB is a fully managed NoSQL database on AWS. It supports key-value and document data models.

Think of it as a managed lookup service:
1. The application provides a key.
2. DynamoDB locates the matching item.
3. AWS manages servers, storage partitions, and replication.

Typical uses include shopping carts, user sessions, product metadata, gaming profiles, IoT state, and high-scale application lookups.

**Why use it?**
- No database servers to install, patch, or manage.
- Designed for high-scale reads and writes.
- Flexible non-key attributes.
- Data is replicated across multiple Availability Zones in a Region.
- Integrates with other AWS services.

**Trade-off:** DynamoDB is not designed for flexible relational joins and arbitrary SQL queries. Understand the application's access patterns before designing keys and indexes.

**Remember:** DynamoDB trades query flexibility for scalable, key-based access.

## 2. Table, Item, Attribute

Think of a spreadsheet:

| DynamoDB | Similar idea | Example |
|---|---|---|
| Table | Collection | Users |
| Item | Record | One user |
| Attribute | Field | Name, Email, City |

Example item:

    {
      "UserId": "U101",
      "Name": "Asha",
      "City": "Pune",
      "Age": 30
    }

Another item in the same table may have different non-key attributes:

    {
      "UserId": "U102",
      "Name": "Ravi",
      "Membership": "Gold"
    }

The key attributes must follow the table's key definition. Other attributes do not all need to be defined in advance.

**Remember:** Table = collection, Item = record, Attribute = field.

## 3. Primary key: the most important concept

Every item must have a primary key that uniquely identifies it. DynamoDB supports two designs.

### A. Simple primary key

Contains only a partition key.

| UserId (partition key) | Name | City |
|---|---|---|
| U101 | Asha | Pune |
| U102 | Ravi | Mumbai |
| U103 | Neha | Hyderabad |

Each UserId must be unique.

### B. Composite primary key

Contains two attributes:

- **Partition key:** groups related items and helps DynamoDB determine where data is stored.
- **Sort key:** distinguishes items in that group and supports ordered/range queries.

Example: an Orders table.

| UserId (partition key) | OrderDate (sort key) | OrderId | Amount |
|---|---|---|---:|
| U101 | 2026-11-01T10:00 | O501 | 1200 |
| U101 | 2026-11-03T12:30 | O508 | 800 |
| U101 | 2026-11-06T09:15 | O519 | 1500 |
| U102 | 2026-11-02T14:00 | O505 | 600 |

    Partition key = UserId
    Sort key      = OrderDate

    U101
      ├── 2026-11-01 -> Order O501
      ├── 2026-11-03 -> Order O508
      └── 2026-11-06 -> Order O519

    U102
      └── 2026-11-02 -> Order O505

Now the application can retrieve all orders for U101, or only U101's orders within a date range.

**Easy memory trick:**
- Partition key = **which group?**
- Sort key = **which item / what order inside the group?**

The combination of partition key and sort key must be unique. Multiple items may share a partition key if their sort-key values differ.

## 4. How DynamoDB distributes data

You design logical keys; DynamoDB manages physical storage partitions.

At a high level:
1. DynamoDB takes the partition-key value.
2. It applies an internal hash function.
3. The result helps determine the physical partition.
4. Items with the same partition-key value are grouped together and ordered by sort key.

    Application writes item
            |
       Read partition key
            |
      Internal hash function
            |
     Choose physical partition
            |
         Store item

You do not manage the physical partitions yourself.

## 5. Hot key / hot partition

Imagine a ticket-booking system. If every booking for a popular concert uses the same partition-key value, EventId = CONCERT-1, a large amount of traffic targets that key. This can create a **hot key / hot partition**.

The table may have unused capacity elsewhere, but one key can still become a bottleneck.

### How to reduce it: write sharding

Spread writes across buckets:

    CONCERT-1#0
    CONCERT-1#1
    CONCERT-1#2
    CONCERT-1#3

The application chooses a bucket, for example using a hash of the booking ID. To retrieve all bookings for the concert, it queries the buckets and combines the results.

This improves write distribution but makes reads and aggregation more complicated.

**Remember:** Good partition key = spreads traffic. Bad partition key = concentrates traffic.

## 6. Query vs Scan

### Query
Reads items using a specific partition-key value. You can also add a condition on the sort key.

Example: "Get all orders for UserId U101."

### Scan
Reads across the table or index and then applies filters.

Example: "Find every order whose Status is PENDING."

| Query | Scan |
|---|---|
| Uses a partition-key value | Reads across the table/index |
| Can use sort-key conditions | Can filter after reading |
| Usually efficient for known access patterns | Can consume significant read capacity |
| Preferred for normal application lookups | Use selectively |

**Remember:** Query = go to the address. Scan = search through the shelves.

A FilterExpression does not make a Scan equivalent to a key-based Query; filtering does not avoid the underlying read work.

## 7. Secondary indexes: another way to query

Suppose the Orders table uses:
- Partition key: UserId
- Sort key: OrderDate

This supports "find orders for a user." What if the application needs "find all PENDING orders"?

A secondary index gives another key path.

### Global Secondary Index (GSI)

A GSI can use a different partition key and sort key from the base table.

    Base table:
      Partition key = UserId
      Sort key      = OrderDate

    GSI:
      Partition key = Status
      Sort key      = OrderDate

Now the application can query pending orders through the GSI.

**Remember:** GSI = different key path across the table.

- A GSI is maintained when base-table items change.
- It can be added after table creation.
- GSI reads are eventually consistent.
- Indexes add storage and write-maintenance cost.

### Local Secondary Index (LSI)

An LSI uses the **same partition key** as the base table but a different sort key.

    Base table:
      Partition key = UserId
      Sort key      = OrderDate

    LSI:
      Partition key = UserId
      Sort key      = Amount

**Remember:** LSI = same group, different sorting.

| GSI | LSI |
|---|---|
| Different partition key allowed | Same partition key as base table |
| Can use a different sort key | Uses a different sort key |
| Can be added after table creation | Must be defined when table is created |
| Eventually consistent reads | Strong or eventual reads |

Use a GSI for a new grouping/query path. Use an LSI for another sort order within the same partition-key group. Do not create indexes for every attribute.

## 8. Read consistency

### Eventually consistent read
A read may briefly return an older value after a write. Replicas catch up and the latest value becomes visible. Use when a short delay is acceptable.

### Strongly consistent read
Returns the latest successful write before the read began, subject to DynamoDB's consistency guarantees. Use when freshness matters.

| Eventual | Strong |
|---|---|
| May briefly return older data | Returns latest committed value |
| Lower read cost | Higher read cost |
| Available for tables, LSIs, GSIs | Not supported for GSIs |

**Remember:** Eventual = may be a little behind. Strong = latest value.

## 9. Capacity: RCU and WCU

- **RCU** = Read Capacity Unit.
- **WCU** = Write Capacity Unit.

Simple rules for standard, non-transactional operations:

### Read
For an item up to 4 KB:
- 1 strongly consistent read per second = 1 RCU.
- 2 eventually consistent reads per second = 1 RCU.

### Write
For an item up to 1 KB:
- 1 write per second = 1 WCU.

Larger items use more capacity, rounded up to the next 4 KB read block or 1 KB write block.

### Easy examples

For a 2 KB item:
- Strong read: rounds to 4 KB = 1 RCU per read/second.
- Eventual read: half the strong-read capacity = 0.5 RCU per read/second.
- Write: rounds to 2 KB = 2 WCUs per write/second.

For a 5 KB item:
- Strong read: rounds to 8 KB = 2 RCUs per read/second.
- Eventual read: half that amount.
- Write: rounds to 5 KB = 5 WCUs per write/second.

Transactional operations consume more capacity than standard operations.

## 10. On-demand vs Provisioned capacity

### On-demand
DynamoDB manages capacity in response to traffic. Billing is based on requests. Good for unpredictable traffic, new applications, and simpler capacity management.

### Provisioned
You specify expected read and write capacity. Auto scaling can adjust it as traffic changes. Good for predictable workloads where capacity can be estimated and managed.

| On-demand | Provisioned |
|---|---|
| Managed in response to requests | You set read/write capacity |
| Good for variable traffic | Good for predictable traffic |
| Pay per request units | Pay for configured capacity |
| Simple starting point | Needs capacity planning |

**Memory trick:** On-demand = let AWS handle request capacity. Provisioned = plan the capacity.

## 11. DynamoDB architecture at a high level

    Application servers
            |
       AWS SDK / API
            |
     DynamoDB managed service
       /       |        \
    Part 1   Part 2    Part 3
       |
       +-- Optional secondary indexes
       +-- Optional DynamoDB Streams
       +-- Optional DAX cache
       +-- Optional Global Tables

The partition key helps route requests. DynamoDB manages physical partitions and replication. Indexes support alternate queries; optional services add caching, change events, and multi-Region replication.

## 12. DynamoDB Streams

Streams records item changes. Think of it as a **change log**.

Example:
1. A customer registers.
2. DynamoDB stores the customer.
3. Streams records the change.
4. AWS Lambda reads the event.
5. Lambda sends a welcome email.

    Application -> DynamoDB -> Stream -> Lambda -> Email service

Uses: notifications, search-index updates, background processing, and derived views. Stream records are retained for 24 hours.

**Remember:** Table stores current state. Stream tells you what changed.

## 13. TTL (Time To Live)

TTL automatically expires items based on a timestamp attribute.

Example: a verification code should expire after 10 minutes.

    {
      "CodeId": "C100",
      "Code": "829104",
      "ExpiresAt": 1794160800
    }

DynamoDB deletes expired items asynchronously. Do not assume deletion happens at the exact expiry second.

Uses: temporary codes, short-lived sessions, and expired application records.

**Remember:** TTL = automatic cleanup after an expiry time.

## 14. DAX: DynamoDB Accelerator

DAX is a managed in-memory cache designed for DynamoDB.

    Application -> DAX cache -> Result (cache hit)
                             -> DynamoDB (cache miss)

It can reduce read latency when the same data is requested frequently.

- DAX is a cache, not the durable database.
- Mainly useful for read-heavy workloads.
- Adds another component and cache behavior.
- Not needed for every DynamoDB application.

**Remember:** DAX = fast memory layer in front of DynamoDB.

## 15. Transactions and conditional writes

DynamoDB supports ACID transactions for coordinating changes to multiple items.

Example: transfer ₹500:
1. Subtract ₹500 from Account A.
2. Add ₹500 to Account B.

Both changes should succeed together, or neither should happen. A transaction provides this all-or-nothing behavior.

DynamoDB also supports conditional writes:
- Update inventory only if AvailableQuantity is greater than zero.
- Create a username only if it does not already exist.

**Remember:** Transaction = several changes succeed together. Condition = change only if a rule is true.

## 16. Global Tables: multiple AWS Regions

A Global Table replicates table data across selected AWS Regions.

    Global Table
       /       \
    Region A  Region B
       \       /
     Replicated data

Useful for users in multiple regions and regional resilience. Multi-Region replication introduces conflict and consistency considerations. Do not assume a cross-Region transaction behaves like a single-Region ACID transaction.

**Remember:** Global Tables = replicas in multiple Regions.

## 17. Item size limit

A DynamoDB item can be up to **400 KB**.

For a large file:
- Store the actual file in Amazon S3.
- Store its S3 key/location and metadata in DynamoDB.

    {
      "DocumentId": "DOC100",
      "OwnerId": "U101",
      "FileName": "report.pdf",
      "S3Key": "reports/2026/report.pdf"
    }

**Remember:** DynamoDB stores the reference and metadata; S3 stores the large object.

## 18. When should we choose DynamoDB?

### Good fit
- High-scale key-value lookups.
- Predictable application access patterns.
- Shopping carts, sessions, user preferences.
- Gaming profiles, event metadata, IoT state.
- Managed scaling and AWS-native applications.

### Not the first choice
- Complex joins across many entities.
- Ad-hoc queries on many unrelated attributes.
- Relational reporting and analytics.
- Workloads where access patterns are unknown and change frequently.
- Applications that depend on flexible SQL queries.

Database choice should follow data shape and access patterns, not simply the "NoSQL" label.

## 19. DynamoDB vs SQL vs Redis

| Technology | Think of it as | Typical use |
|---|---|---|
| DynamoDB | Managed scalable key-value/document DB | High-scale key-based access |
| PostgreSQL / MySQL | Relational database with SQL | Joins, constraints, flexible queries |
| Redis | In-memory data store | Cache, counters, sessions |

Simple selection:
- Need joins and flexible SQL? Consider PostgreSQL/MySQL.
- Need very fast temporary/cache data? Consider Redis.
- Need managed, scalable access around known keys? Consider DynamoDB.

They can also work together; they are not always mutually exclusive.

## 20. Common interview questions

**Q1. What is DynamoDB?**  
A fully managed AWS NoSQL key-value and document database designed for scalable, low-latency access patterns.

**Q2. What is a partition key?**  
The key attribute DynamoDB hashes to help determine where an item is stored.

**Q3. What is a sort key?**  
An optional second key that orders and distinguishes items sharing a partition-key value.

**Q4. What is a hot partition?**  
A partition receiving disproportionate traffic, often because many requests use the same partition-key value.

**Q5. Query vs Scan?**  
Query uses a partition-key value. Scan reads across the table or index and can be expensive.

**Q6. GSI vs LSI?**  
GSI can use a different partition key. LSI keeps the same partition key but changes the sort key.

**Q7. On-demand vs Provisioned?**  
On-demand suits variable traffic. Provisioned suits workloads where capacity can be planned.

**Q8. What are DynamoDB Streams?**  
A time-ordered record of item changes for downstream processing.

**Q9. What is TTL?**  
A feature that asynchronously deletes items after a configured expiry timestamp.

**Q10. What is DAX?**  
A managed in-memory cache for DynamoDB reads.

**Q11. How do you avoid hot keys?**  
Choose high-cardinality partition keys, distribute activity, and use write sharding when necessary.

**Q12. Why not use DynamoDB for every application?**  
It is optimized for key-based access, not arbitrary relational joins and ad-hoc queries.

## 21. One-minute revision

| Concept | Remember it as |
|---|---|
| DynamoDB | Managed NoSQL database on AWS |
| Table / Item / Attribute | Collection / record / field |
| Partition key | Which group / routing key |
| Sort key | Order within the group |
| Physical partition | Storage managed by AWS |
| Hot key | Too much traffic for one key |
| Query | Find by partition key |
| Scan | Read across the table |
| GSI | Different partition-key query path |
| LSI | Same partition key, different sort key |
| RCU / WCU | Read / write capacity |
| On-demand | Capacity responds to requests |
| Provisioned | Capacity planned by you |
| Streams | Record of changes |
| TTL | Expire old items |
| DAX | Cache for DynamoDB |
| Global Tables | Replicas across Regions |
| Transaction | Several changes succeed together |

## 22. How to explain DynamoDB in an interview

Use this sequence:

1. **Workload:** Why does the application need scalable key-value/document access?
2. **Access patterns:** What are the important reads and writes?
3. **Keys:** Explain partition key and sort key with an example.
4. **Distribution:** Explain hashing, partitions, and hot-key prevention.
5. **Queries:** Explain Query vs Scan and the indexes required.
6. **Operations:** Mention capacity mode, consistency, transactions, and TTL where relevant.
7. **Trade-offs:** Explain what DynamoDB makes easy and what becomes harder, such as joins and ad-hoc queries.

A strong answer is not "DynamoDB scales automatically." Explain **how access patterns shaped the key design, how traffic is distributed, and what trade-offs the design makes**.

## 23. References

- [AlgoMaster – DynamoDB Deep Dive](https://algomaster.io/learn/system-design-interviews/dynamodb) – supplied learning reference; detailed chapter content requires premium access.
- [AWS – DynamoDB core components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html)
- [AWS – Partition key design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)
- [AWS – DynamoDB overview and design guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-rdbms-dynamodb/overview.html)

---

**Final memory line:** DynamoDB design starts with the questions your application asks. Choose keys that answer those questions efficiently, distribute traffic evenly, and add indexes only for the query patterns you actually need.
