# Understanding the Catalog of the iceberg and maintaining the table
## Problem Point
As a data engineer,Once we create a iceberg table.When the update,insert,delete operation has been happened,
not only should we maintain the table,but also we should know why do this.So we need to understand the change of 
iceberg catalog when some operation has been happened.
## Explanation
### Create table 
When we create a iceberg table,the iceberg background will create a catalog for us.For example:
```sql
CREATE TABLE test_table (
  id INT,
  name STRING,
  score INT
) USING iceberg;
``` 
next,hdfs dfs -ls xxx/warehouse/db/test_table
```plaintext
hdfs://xxx/warehouse/db/test_table/   
├── data/           
└── metadata/
    └── 0000-xxx-.metadata.json  
```
### Insert data
when we insert data into the table,the catalog will change as follows:
```plaintext
hdfs://xxx/warehouse/db/test_table/
├── data/
│   └── 00000-xxxx-xxxx-xxxx.parquet  # new added：data file,contains the actual data
└── metadata/
    ├── xxx.metadata.json        # v1：empty metadata during create table（snapshots is empty）
    ├── xxx.metadata-v2.json     # v2：this commit generate the current effective metadata file
    ├── snap.xxxx.m1.avro              # manifest-list avro,this file contains the list of manifests
    └── xxx.m2.avro              # manifest avro,this file records the list of data files
```
the metadata.json file contains the following information:
```json
{
  "format-version": 2,
  "table-uuid": "87a00199-xxxx-xxxx-xxxx-2f8c98211234",
  "location": "hdfs://xxx/warehouse/default/test_table",
  "last-updated-ms": 1760300000000,
  "last-column-id": 3,
  "schema": {
    "type": "struct",
    "schema-id": 0,
    "fields": [
      {"id":1,"name":"id","type":"int","required":false},
      {"id":2,"name":"name","type":"string","required":false},
      {"id":3,"name":"score","type":"int","required":false}
    ]
  },
  "partition-specs": [{"spec-id":0,"fields":[]}],
  "default-spec-id":0,
  "sort-orders": [{"order-id":0,"fields":[]}],
  "default-sort-order-id":0,
  "properties": {},
  "current-snapshot-id": 5689012345678901234,
  "snapshots": [
    {
      "snapshot-id": 5689012345678901234,
      "parent-snapshot-id": null,
      "sequence-number":1,
      "timestamp-ms":1760300000000,
      "operation": "append",
      "manifest-list": "hdfs://xxx/warehouse/default/test_table/metadata/1234-manifest-list.avro",
      "summary": {
        "added-data-files": "1",
        "added-records": "1",
        "added-files-size": "2456",
        "changed-partition-count": "1",
        "total-records": "1",
        "total-data-files": "1",
        "total-file-size": "2456"
      }
    }
  ],
  "snapshot-log": [
    {
      "timestamp-ms":1760300000000,
      "snapshot-id":5689012345678901234
    }
  ],
  "metadata-log": [
    {
      "timestamp-ms": 1760299000000,
      "metadata-file": "hdfs://xxx/warehouse/default/test_table/metadata/metadata-v1.json"
    },
    {
      "timestamp-ms": 1760300000000,
      "metadata-file": "hdfs://xxx/warehouse/default/test_table/metadata/metadata-v2.json"
    }
  ]
}
```
the file metadata.json mainly contains the following information:
1. table-uuid: unique identifier for the table 
2. location: location of the table
3. schema: schema of the table, including fields, their types, and whether they are required or not
4. partition-specs: partitioning specification of the table, including the fields and the order of the fields
5. current-snapshot-id: the id of the current snapshot
6. snapshots: a list of snapshots, each snapshot contains information about the changes made to the table
7. manifest-list: a list of manifests, each manifest contains information about the data files, including their locations, sizes, and other metadata

### Update data
When we update data in the table, the catalog will change as follows:
```plaintext
test_table/
├── data/
│   └── xxx.parquet       # old parquet（original id=1 score=90，physical file retained, current snapshot no longer references）
│   └── yyy.parquet       # ✅【new added】update rewrite parquet, id=1 score=95
└── metadata/
    ├── metadata-v1.json        # build table version, retained
    ├── metadata-v2.json        # insert version, retained (snapshot_2)
    ├── metadata-v3.json        # ✅【new added】this commit generated, **current effective metadata**
    ├── 1234-manifest-list.avro # snapshot2's manifest-list (insert that time, retained)
    ├── 1234-manifest.avro      # snapshot2's manifest, retained
    ├── 5678-manifest-list.avro # ✅【new added】snapshot3's manifest-list
    └── 5678-manifest.avro      # ✅【new added】snapshot3's manifest
```
the metadata.json file key changes(vs v2), including:
```json
{
  "format-version": 2,
  "table-uuid": "87a00199-xxxx-xxxx-xxxx-2f8c98211234",
  "current-snapshot-id": 7894561230123456789,
  "snapshots": [
    {
      "snapshot-id": 5689012345678901234, 
      "parent-snapshot-id": null,
      "timestamp-ms":1760300000000,
      "operation": "append",
      "manifest-list": "hdfs://xxx/test_table/metadata/1234-manifest-list.avro",
      "summary": {
        "added-data-files": "1",
        "added-records": "1",
        "total-records":"1"
      }
    },
    {
      "snapshot-id": 7894561230123456789,
      "parent-snapshot-id": 5689012345678901234,
      "sequence-number":2,
      "timestamp-ms":1760300100000,
      "operation": "overwrite",
      "manifest-list": "hdfs://xxx/test_table/metadata/5678-manifest-list.avro",
      "summary": {
        "added-data-files": "1",
        "deleted-data-files": "1",
        "added-records": "1",
        "deleted-records": "1",
        "total-records": "1",
        "total-data-files": "1"
      }
    }
  ],
  "snapshot-log": [
    {"timestamp-ms":1760300000000,"snapshot-id":5689012345678901234},
    {"timestamp-ms":1760300100000,"snapshot-id":7894561230123456789}
  ],
  "metadata-log": [
    {"timestamp-ms":1760299000000,"metadata-file":"hdfs://xxx/test_table/metadata/metadata-v1.json"},
    {"timestamp-ms":1760300000000,"metadata-file":"hdfs://xxx/test_table/metadata/metadata-v2.json"},
    {"timestamp-ms":1760300100000,"metadata-file":"hdfs://xxx/test_table/metadata/metadata-v3.json"}
  ]
}
```
change points:
1. current-snapshot-id: updated to the new snapshot ID
2. snapshots: a new snapshot is added, containing information about the update operation
3. manifest-list: a new manifest-list is added, containing information about the manifests created during the update operation
4. manifest: a new manifest is added, containing information about the data files affected by the update operation
5. metadata-log: a new entry is added, containing information about the new metadata file
Data reading link (currently, the table reading uses v3)
metadata-v3.json → current-snapshot-id = snapshot3 → snapshot3 的 manifest-list (5678) → manifest (5678) → yyy.parquet
### recovery data
If we find some data is wrong, we can recover it by using the snapshot ID.
```sql
// query the snapshot ID
SELECT * FROM test_table.snapshots;
//then we can recover the data
call system.recover_table
('test_table',
 5689012345678901234);
```
When we do this operation, the table will switch to the specified snapshot ID, and all subsequent snapshots will be rolled back.
### Table Maintenance
With the operation of update,delete,insert,we find that the data file and metadata file will increase.
At first, we should merge small files to reduce the number of data files.
```sql
call system.rewrite_table_files(
table =>'test_table');
```
Secondly, we should clean expired snapshots and metadata files to save storage space.
```sql
call system.expire_snapshots(
table =>'test_table',
older_than => timestamp '2026-10-01 00:00:00');
// or 
call system.expire_snapshots(
table ==>'test_table',
retain_last => 5);
```
Thirdly, we should remove orphan files to save storage space.
```sql
call system.remove_orphan_files(
table =>'test_table');
```

Thank you for reading, if you have any questions, welcome to exchange and discuss.


















