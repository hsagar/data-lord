PySpark Interview – DataFrame-Only Cheat-Sheet 

--------------------------------------------------------
0. 60-sec boot-strap (run once)
--------------------------------------------------------
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from pyspark.sql.window import Window

spark = (SparkSession.builder
         .master("local[*]")
         .appName("interview-notes")
         .config("spark.sql.adaptive.enabled","true")
         .getOrCreate())

spark.sparkContext.setLogLevel("WARN")
```

--------------------------------------------------------
1. Creating DataFrames (5 ways you must know)
--------------------------------------------------------
```python
# 1. Range
df0 = spark.range(0, 10, 2)          # id:0,2,4…

# 2. Python objects
data = [(1, "apple", 3.0), (2, "banana", 5.5)]
schema = StructType([StructField("id",IntegerType()),
                     StructField("fruit",StringType()),
                     StructField("weight",DoubleType())])
df1 = spark.createDataFrame(data, schema)

# 3. CSV (header & inferred schema)
# df2 = spark.read.option("header","true").option("inferSchema","true").csv("file.csv")

# 4. JSON
jsonRows = [ '{"id":1, "fruit":"apple"}', '{"id":2, "fruit":"kiwi"}' ]
df3 = spark.read.json(spark.sparkContext.parallelize(jsonRows))

# 5. SQL
spark.sql("SELECT 1 as id, 'spark' as word").show()
```

--------------------------------------------------------
2. Schema & Columns
--------------------------------------------------------
```python
df1.printSchema()
df1.columns                           # list of names
df1.select("id").dtypes               # [('id', 'int')]
df1 = df1.withColumnRenamed("fruit","item")
df1 = df1.drop("weight")
```

--------------------------------------------------------
3. SELECT / WITHCOLUMN / TRANSFORM
--------------------------------------------------------
```python
# selectExpr = SQL expressions
df1.selectExpr("id*2 as double_id", "upper(item) as UP").show()

# withColumn: add or replace
df1 = df1.withColumn("kg", col("weight")*0.45)\
         .withColumn("category", lit("fresh"))

# transform array<string> column (Spark 3+)
dfArr = spark.createDataFrame([([1,2,3],)], ["arr"])
dfArr.select(transform(col("arr"), lambda x: x*2).alias("doubled")).show()
```

--------------------------------------------------------
4. Filtering
--------------------------------------------------------
```python
df1.filter( (col("weight") > 4) & (col("item").like("%a%")) ).show()
df1.where("weight > 4 AND item like '%a%'").show()   # SQL string
```

--------------------------------------------------------
5. Aggregations (groupBy / agg / pivot)
--------------------------------------------------------
```python
sales = [(2020,"Q1","phones",1000),
         (2020,"Q1","tabs",  400),
         (2020,"Q2","phones",1100),
         (2021,"Q1","phones",1500)]
salesDF = spark.createDataFrame(sales, ["year","qtr","product","amount"])

# basic
salesDF.groupBy("year").sum("amount").show()

# multi-agg
salesDF.groupBy("year")\
       .agg(count("*").alias("rows"),
            avg("amount").alias("avg_amt"),
            collect_list("product").alias("prods"))\
       .show()

# pivot
pv = salesDF.groupBy("year")\
            .pivot("qtr", ["Q1","Q2"])\
            .sum("amount")
pv.show()
# +----+----+----+
# |year|  Q1|  Q2|
# +----+----+----+
# |2020|1400|1100|
# |2021|1500|null|
# +----+----+----+
```

--------------------------------------------------------
6. Joins (broadcast, salted, range)
--------------------------------------------------------
```python
cust = spark.createDataFrame([(1,"Ann"),(2,"Bob"),(3,"Cat")], ["cid","name"])
ord  = spark.createDataFrame([(101,1,50),(102,2,30),(103,99,10)], ["oid","cid","value"])

# inner
cust.join(ord, "cid").show()

# left anti (customers with NO orders)
cust.join(ord, "cid", "left_anti").show()

# broadcast hint
from pyspark.sql.functions import broadcast
bigFact.join(broadcast(smallDim), "key")
```

--------------------------------------------------------
7. Window functions (row_number, rank, lag, running total)
--------------------------------------------------------
```python
win = Window.partitionBy("product").orderBy("year")

salesDF.withColumn("rn", row_number().over(win))\
       .withColumn("prev", lag("amount",1).over(win))\
       .withColumn("cumsum", sum("amount").over(win.rowsBetween(Window.unboundedPreceding, 0)))\
       .show()
```

--------------------------------------------------------
8. Null handling
--------------------------------------------------------
```python
df = spark.createDataFrame([(1,None),(2,20)], ["id","score"])
df.na.drop(subset=["score"])\
  .na.fill({"score":0})\
  .select(coalesce(col("score"),lit(0))).show()
```

--------------------------------------------------------
9. Duplicates
--------------------------------------------------------
```python
df.distinct()
df.dropDuplicates(["year","qtr"])
```

--------------------------------------------------------
10. Array & Map columns (explode, inline, higher-order)
--------------------------------------------------------
```python
dfArr = spark.createDataFrame([("a",[1,2,3],{"x":10})], ["name","nums","meta"])
dfArr.select("name", explode("nums").alias("num")).show()
dfArr.select("name", posexplode("nums")).show()  # index,value
dfArr.select("name", expr("inline(arrays_zip(nums,transform(nums,x->x*2)))")).show()
```

--------------------------------------------------------
11. JSON column manipulation
--------------------------------------------------------
```python
j = '{"person":{"name":"Mike","age":30}}'
dfJ = spark.createDataFrame([(j,)], ["jsonCol"])
dfJ.select(get_json_object("jsonCol","$.person.name").alias("name"),
           json_tuple("jsonCol","person")).show()  # json_tuple returns one column
```

--------------------------------------------------------
12. UDF vs pandas UDF (vectorized)
--------------------------------------------------------
```python
# regular UDF
def tax(rate): return rate*0.2
taxUDF = udf(tax, DoubleType())
df1.withColumn("tax", taxUDF("weight")).show()

# pandas UDF (vectorized) – Spark 3+
from pyspark.sql.functions import pandas_udf
@pandas_udf("double")
def tax_pd(s: pd.Series) -> pd.Series: return s*0.2
df1.withColumn("tax", tax_pd("weight")).show()
```

--------------------------------------------------------
13. Repartition vs Coalesce
--------------------------------------------------------
```python
df.rdd.getNumPartitions()
df = df.repartition(200, "year")   # full shuffle, can partition by column
df = df.coalesce(10)               # avoids shuffle, only decrease
```

--------------------------------------------------------
14. Caching & Check-pointing
--------------------------------------------------------
```python
df.cache()                         # lazy, MEMORY_AND_DISK by default
df.persist(StorageLevel.MEMORY_ONLY_2)
df.count()                         # action triggers cache
df.checkpoint()                    # cuts lineage, saves to HDFS/tmp
```

--------------------------------------------------------
15. Writing data (parquet, partitionBy, bucketBy, append/overwrite)
--------------------------------------------------------
```python
(df.write.mode("overwrite")
   .partitionBy("year","qtr")
   .parquet("s3://bucket/sales_parquet"))

# bucket (fixed # files per partition)
(df.write.mode("overwrite")
   .bucketBy(16, "cid")
   .sortBy("cid")
   .saveAsTable("bucketed_orders"))
```

--------------------------------------------------------
16. Performance tuning checklist (interview bullets)
--------------------------------------------------------
- Use columnar formats (parquet) – predicate push-down & column pruning  
- Partition by low-cardinality, high-filter columns  
- Bucket on high-frequency join keys (avoids shuffle)  
- Filter early, select only needed columns  
- Broadcast joins if one side < 30 MB (spark.sql.autoBroadcastJoinThreshold)  
- Avoid UDFs – prefer Spark built-ins; if needed use pandas UDF  
- Adaptive Query Execution (AQE) – Spark 3+ (spark.sql.adaptive.enabled)  
- Monitor: Spark UI → SQL tab → check “WholeStageCodegen”, spill, skew  
- Salting for skewed keys: add salted column, join, then remove salt  
- Coalesce before small-file write to avoid tiny files  
- Cache only when DF is reused ≥ 2× and fits memory  

--------------------------------------------------------
17. Mini-project (end-to-end 40 lines)
--------------------------------------------------------
Problem: compute revenue running total per customer, flag first 3 orders, write partitioned parquet.

```python
orders = [(1, 100, "2022-01-01", 50),
          (1, 101, "2022-01-05", 30),
          (1, 102, "2022-01-10", 20),
          (2, 200, "2022-02-01", 70),
          (2, 201, "2022-02-02", 40)]
ordDF = spark.createDataFrame(orders, ["cust_id","order_id","date","amt"])

win = Window.partitionBy("cust_id").orderBy("date")

final = (ordDF.withColumn("running", sum("amt").over(win))
              .withColumn("rn", row_number().over(win))
              .withColumn("flag", when(col("rn") <= 3, "newbie").otherwise("regular"))
              .withColumn("year", year(col("date"))))

(final.write.mode("overwrite")
      .partitionBy("year","cust_id")
      .parquet("/tmp/orders_enriched"))
```

--------------------------------------------------------
18. Quick memory cards (one-liners you’ll be asked)
--------------------------------------------------------
- `selectExpr("stack(3, 'a',1,'b',2,'c',3) as (letter,num)")` – unpivot  
- `arrays_overlap(col("a"),col("b"))` – returns boolean  
- `regexp_extract("price_$100", r"_\\$(\\d+)", 1)` – capture group  
- `date_trunc("week", col("ts"))` – Monday 00:00  
- `percentile_approx("salary", 0.5, 100)` – median with accuracy  
- `df.summary("count","min","25%","75%","max")` – five-num summary  
- `spark.sql("SET spark.sql.shuffle.partitions=200")` – runtime config  
- `spark.catalog.listTables()` – see temp views  
- `spark.table("orders").write.saveAsTable("orders_hive")` – permanent table  

--------------------------------------------------------
19. Common interview coding questions (copy-paste solutions)
--------------------------------------------------------
Q1. Top 3 salaries per department  
```python
win = Window.partitionBy("dept").orderBy(desc("salary"))
empDF.withColumn("rn", dense_rank().over(win)).filter("rn <= 3").show()
```

Q2. Second highest salary overall  
```python
empDF.orderBy(desc("salary")).limit(2).orderBy("salary").limit(1).show()
# or
empDF.agg(max(when(col("salary") != col("max_sal"), col("salary"))).alias("2nd"))\
     .crossJoin(empDF.agg(max("salary").alias("max_sal"))).show()
```

Q3. Convert array<map> to rows  
```python
df = spark.createDataFrame([([({"k":1},{"k":2})],)], ["arr"])
df.select(inline(col("arr"))).show()
```

Q4. Word count on DataFrame column  
```python
df = spark.createDataFrame([("hello world hello",)], ["line"])
df.select(explode(split("line"," ")).alias("word")).groupBy("word").count().show()
```

--------------------------------------------------------
20. Final 30-sec revision slide (say it in the interview)
--------------------------------------------------------
“Spark DataFrame is an immutable distributed collection of Rows with a known Schema.  
All transformations are lazy; actions trigger the DAG.  
Narrow ops (map, filter) stay in stage; wide ops (groupBy, join) cause shuffle.  
I cache when reused, partition by filter columns, bucket on join keys, prefer built-ins over UDFs, and let Adaptive Query Execution fix skew at runtime.”

--------------------------------------------------------
That’s it—one single doc, zero external tabs needed.  
Good luck in your interview!
