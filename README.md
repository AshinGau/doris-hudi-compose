# Doris+Hudi+MINIO Environments
Launch spark/doris/hive/hudi/minio test environments, and give examples to query hudi in Doris.

## Launch Docker Compose
**Launch all components in docker**
``` shell
sudo ./start-hudi-compose.sh
```
**Login into Spark**
```shell
sudo ./login-spark.sh
```
**Login into Doris**
```shell
sudo ./login-doris.sh
```

## Prepare Hudi Data
There's already a hive table named `customer` in hive default. Create a hudi table from the hive table:
``` sql
-- ./login-spark.sh
use default;

-- create a COW table
CREATE TABLE customer_cow
USING hudi
TBLPROPERTIES (
  type = 'cow',
  primaryKey = 'c_custkey',
  preCombineField = 'c_name'
)
PARTITIONED BY (c_nationkey)
AS SELECT * FROM customer;

-- create a MOR table
CREATE TABLE customer_mor
USING hudi
TBLPROPERTIES (
  type = 'mor',
  primaryKey = 'c_custkey',
  preCombineField = 'c_name'
)
PARTITIONED BY (c_nationkey)
AS SELECT * FROM customer;
```

## Query Data
Doris refresh hive catalog in [10min in default](https://doris.apache.org/docs/lakehouse/datalake-analytics/hive/#metadata-cache--refresh),
users can refresh directly to access the hudi table in Doris by `doris> refresh catalog hive;`

After hudi table is ready in Doris, all operations in hudi table will be detected by Doris, and there's no need to refresh catalog or tables.

Insert new data into hudi tables in spark-sql:
```sql
spark-sql> insert into customer_cow values (100, "Customer#000000100", "jD2xZzi", "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests", 25);
spark-sql> insert into customer_mor values (100, "Customer#000000100", "jD2xZzi", "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests", 25);
```
`c_nationkey=25` is a new partition, doris can query the new data at once without refresh:
```
doris> select * from customer_cow where c_custkey = 100;
doris> select * from customer_mor where c_custkey = 100;
```
Insert a record with `c_custkey=32`(primary key, already in table) will remove the old record:
```
spark-sql> insert into customer_cow values (32, "Customer#000000032_update", "jD2xZzi", "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests", 15);
spark-sql> insert into customer_mor values (32, "Customer#000000032_update", "jD2xZzi", "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests", 15);
```
Query the updated data at once in doris:
```
doris> select * from customer_cow where c_custkey = 32;
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
| c_custkey | c_name                    | c_address | c_nationkey | c_phone         | c_acctbal | c_mktsegment | c_comment                           |
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
|        32 | Customer#000000032_update | jD2xZzi   |          15 | 25-430-914-2194 |   3471.59 | BUILDING     | cial ideas. final, furious requests |
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
doris> select * from customer_mor where c_custkey = 32;
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
| c_custkey | c_name                    | c_address | c_nationkey | c_phone         | c_acctbal | c_mktsegment | c_comment                           |
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
|        32 | Customer#000000032_update | jD2xZzi   |          15 | 25-430-914-2194 |   3471.59 | BUILDING     | cial ideas. final, furious requests |
+-----------+---------------------------+-----------+-------------+-----------------+-----------+--------------+-------------------------------------+
```







