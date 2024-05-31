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
create table customer_cow (
  c_custkey INT,
  c_name VARCHAR(25),
  c_address VARCHAR(40),
  c_nationkey INT,
  c_phone CHAR(15),
  c_acctbal DECIMAL(12,2),
  c_mktsegment CHAR(10),
  c_comment VARCHAR(117))
using hudi
TBLPROPERTIES (
  type = 'cow',
  primaryKey = 'c_custkey'
);

-- create a MOR table
create table customer_mor (
  c_custkey INT,
  c_name VARCHAR(25),
  c_address VARCHAR(40),
  c_nationkey INT,
  c_phone CHAR(15),
  c_acctbal DECIMAL(12,2),
  c_mktsegment CHAR(10),
  c_comment VARCHAR(117))
using hudi
TBLPROPERTIES (
  type = 'mor',
  primaryKey = 'c_custkey'
);
```
Insert data into hudi table:
```sql
spark-sql> insert into customer_cow select * from customer where c_custkey < 320;
spark-sql> insert into customer_mor select * from customer where c_custkey < 320;
```

## Query Data
Doris refresh hive catalog in [10min in default](https://doris.apache.org/docs/lakehouse/datalake-analytics/hive/#metadata-cache--refresh),
users can refresh directly to access the hudi table in Doris by `doris> refresh catalog hive;`

After hudi table is ready in Doris, all operations in hudi table will be detected in Doris, and there's no need to refresh catalog or tables.

Insert new data into hudi tables in spark-sql:
```sql
spark-sql> insert into customer_cow values (1, "Customer#000000001", "jD2xZzi", 15, "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests");
spark-sql> insert into customer_mor values (1, "Customer#000000001", "jD2xZzi", 15, "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests");
```
Query the new data at once in doris:
```
doris> select * from customer_cow where c_custkey = 1;
doris> select * from customer_mor where c_custkey = 1;
```
Insert a record with `c_custkey=32`(primary key, already in table) will remove the old record:
```
spark-sql> insert into customer_cow values (32, "Customer#000000032_update", "jD2xZzi", 15, "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests");
spark-sql> insert into customer_mor values (32, "Customer#000000032_update", "jD2xZzi", 15, "25-430-914-2194", 3471.59, "BUILDING", "cial ideas. final, furious requests");
```


