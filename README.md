# gravity-shard: Brobridge's shard manager

## Shard Manager 
* ShardType: Primary-Only/Secondary-Only/Primary-Secondary 三選一
* ApplicationId: Application 的 UUID
* Size: 這個 Shard Physical 有多少 Pod

### Examples
#### Primary-Only: HA Adapter
``` json
{
  "ShardType": "PrimaryOnly",
  "ApplicationId": "01234567-89ab-cdef-0123-456789abcdef",
  "Size": 2
}
```
#### Secondary-Only: Cache Pool 
``` json
{
  "ShardType": "SecondaryOnly",
  "ApplicationId": "01234567-89ab-cdef-0123-456789abcdef",
  "Size": 5
}
```
#### Primary-Secondary: 3-Shard Key-Value
``` json
{
  "ShardType": "PrimarySecondary",
  "ApplicationId": "01234567-89ab-cdef-0123-456789abcdef",
  "Size": 9
}
```

## Shard Type
### Primary-Only
群集只有唯一的 Shard，意味著 Queue Group 的特性，通常 State 儲存在群集之外，像是寫入、讀取外部 Database。

可使用 Deployment/Statefulset，大小為任意自然數，同時只會有一個 Shard 處理邏輯。

Each shard has a single replica, called the primary replica. These types of applications typically store state in external systems, like databases and data warehouses. A common paradigm is that each shard represents a worker that fetches designated data, processes them, optionally serves client requests, and writes back the results, with optional optimizations like batching. Stream processing is one real example that processes data from an input stream and writes results to an output stream. Shard Manager provides an at-most-one-primary guarantee to help prevent data inconsistency due to duplicate data processing, like a traditional ZooKeeper lock-based approach.

### Secondary-Only
群集所有的副本有同樣地位，不需要考慮副本之間的一致性。

可使用 Deployment/Statefulset，大小為任意自然數，所有的副本可以同時處理。

Each shard has multiple replicas of equal role, dubbed secondaries. The redundancy from multiple replicas provides better fault tolerance. Additionally, replication factor can be adjusted based on workload: Hot shards can have more replicas to spread load. Typically, these types of applications are read-only without strong consistency requirements. They fetch data from an external storage system, optionally process the data, cache results locally, and serve queries off local data. One real example is machine learning inference systems, which download trained models from remote storage and serve inference requests.

### Primary-Secondary
一組 Paxos 為一個 Shard，實際上每一個 Primary-Secondary Shard 為一組 3 nodes Raft。

使用 Statefulset，大小為 3*N。

Each shard has multiple replicas of two roles — primary and secondary. These types of applications are typically storage systems with strict requirements on data consistency and durability, where the primary replica accepts write requests and drives replication among all replicas while secondaries provide redundancy and can optionally serve reads to reduce the load on the primary. One example is Zippy DB, which is a global key-value store with Paxos-based replication.

## Workflow
### Add shards
1. 啟動底層的 Pod Cluster
2. 更新 Shard Manager
3. 更新 Shard Discovery Service

### Client Connection
Client 不需要知道

1. Read Shard Discovery Service
  * 需要服務的是哪一個 Application (帶入環境變數)
  * 到底要連接多少個 Shard (從 Shard Manager 讀取)
  * Keys 要怎麼 Partition 到 Shard 上面 (從 Shard Manager 讀取後計算)
3. 計算 Routing Logic


