# Chapter 2: RDD Operations and Transformations

## 2.1 RDD Fundamentals

### The Nature of RDDs

At its core, an RDD (Resilient Distributed Dataset) is not just a distributed collection - it's a recipe for computing a dataset. Each RDD contains:

1. Partitions: The atomic pieces of the dataset
2. Dependencies: How this RDD relates to its parent RDDs
3. Function: Computation to be performed on parent RDD
4. Partition Scheme: How data is partitioned across nodes
5. Preferred Locations: Data locality preferences

Let's examine each in detail:

#### Partitions Internal Structure
```scala
abstract class RDD[T: ClassTag](
  @transient private var _sc: SparkContext,
  @transient private var deps: Seq[Dependency[_]]
) extends Serializable {
  
  def getPartitions: Array[Partition]  // Define partitioning scheme
  def compute(split: Partition, context: TaskContext): Iterator[T]  // Compute partition
  def getDependencies: Seq[Dependency[_]]  // Define dependencies
  def getPreferredLocations(split: Partition): Seq[String]  // Data locality
}
```

When working with HBase snapshots, the partitioning is particularly interesting:

```scala
class HBaseSnapshotRDD(
  sc: SparkContext,
  snapshotName: String,
  hbaseConf: Configuration
) extends RDD[Result](sc, Nil) {
  
  override def getPartitions: Array[Partition] = {
    // Each HBase region becomes a partition
    val snapshotInputFormat = new TableSnapshotInputFormat
    val jobContext = new JobContextImpl(hbaseConf, new JobID)
    snapshotInputFormat.getSplits(jobContext).map { split =>
      new HBaseSnapshotPartition(split.asInstanceOf[TableSnapshotInputFormat.TableSnapshotRegionSplit])
    }
  }
}
```

### Dependencies and Lineage

RDDs maintain two types of dependencies:

1. Narrow Dependencies:
   - Each partition of the parent RDD is used by at most one partition of the child RDD
   - Allows for pipelined execution
   - Example: map, filter, mapPartitions

```scala
// Narrow transformation example with HBase data
val regionsRDD = hbaseRDD.mapPartitions { iterator =>
  iterator.filter(result => {
    val rowKey = Bytes.toString(result.getRow)
    rowKey.startsWith("2024")  // Filter by row key prefix
  })
}
```

2. Wide Dependencies:
   - Multiple child partitions may depend on each partition of the parent
   - Requires shuffle
   - Example: groupByKey, reduceByKey, join

```scala
// Wide transformation example
val aggregatedData = regionsRDD.map { result =>
  val rowKey = Bytes.toString(result.getRow)
  val value = parseValue(result)
  (rowKey.substring(0, 8), value)  // Group by date
}.reduceByKey(_ + _)  // Triggers shuffle
```

### Execution and Materialization

RDDs are lazy - they don't hold data until an action is called. Understanding this materialization process is crucial:

1. Action Triggers:
```scala
// No computation happens here
val filteredRDD = hbaseRDD.filter(someCondition)
val mappedRDD = filteredRDD.map(transformData)

// Computation starts here
val count = mappedRDD.count()  // Action triggers execution
```

2. Stage Formation:
   - Spark traces dependencies backward from the action
   - Creates stages at shuffle boundaries
   - Optimizes within stages for pipelined execution

```scala
// Single stage example (no shuffle)
hbaseRDD
  .filter(_.getRow.length > 10)
  .map(processRow)
  .foreach(println)

// Multi-stage example (with shuffle)
hbaseRDD
  .map(r => (getKey(r), getValue(r)))
  .reduceByKey(_ + _)  // Stage boundary
  .collect()
```

## 2.2 RDD Operations with HBase Snapshots

### Reading from HBase Snapshots

When creating an RDD from an HBase snapshot, several key operations occur:

1. Partition Creation:
```scala
def createHBaseSnapshotRDD(
  sc: SparkContext, 
  snapshotName: String,
  tableName: String
): RDD[Result] = {
  
  // Configure snapshot scan
  val conf = HBaseConfiguration.create()
  conf.set(TableInputFormat.INPUT_TABLE, tableName)
  conf.set(TableInputFormat.SCAN, convertScanToString(scan))
  
  // Create RDD with HBase partitioning
  sc.newAPIHadoopRDD(
    conf,
    classOf[TableSnapshotInputFormat],
    classOf[ImmutableBytesWritable],
    classOf[Result]
  )
}
```

2. Scan Configuration:
```scala
def configureScan(scan: Scan): Scan = {
  // Optimization configurations
  scan.setCaching(1000)  // Number of rows to fetch per RPC
  scan.setBatch(100)     // Number of columns to fetch per row
  scan.setCacheBlocks(false)  // Disable block cache for full table scan
  
  // Add filters if needed
  val filter = new FilterList(FilterList.Operator.MUST_PASS_ALL)
  filter.addFilter(new FirstKeyOnlyFilter())  // Example filter
  scan.setFilter(filter)
  
  scan
}
```

### Transformation Types and Optimization

#### 1. Row-Level Transformations

These operate on individual HBase Result objects:

```scala
// Basic row transformation
val processedRDD = hbaseRDD.map { result =>
  val rowKey = Bytes.toString(result.getRow)
  val cf = result.getFamiliesMap
  
  // Process columns
  val processedData = for {
    family <- cf.keySet.asScala
    qualifier <- result.getFamilyMap(family).keySet.asScala
    value = result.getValue(family, qualifier)
  } yield {
    // Transform data
    (Bytes.toString(qualifier), Bytes.toString(value))
  }
  
  (rowKey, processedData.toMap)
}
```

#### 2. Partition-Level Transformations

More efficient as they work with entire partitions:

```scala
val optimizedRDD = hbaseRDD.mapPartitions { iterator =>
  // Setup once per partition
  val processor = new DataProcessor()
  
  iterator.map { result =>
    processor.process(result)
  }
}
```

#### 3. Custom Partitioning

For better data distribution:

```scala
class CustomHBasePartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions
  
  override def getPartition(key: Any): Int = {
    val rowKey = key.asInstanceOf[String]
    // Custom logic for partition assignment
    abs(rowKey.hashCode % numPartitions)
  }
}

// Apply custom partitioning
val repartitionedRDD = processedRDD.repartitionAndSortWithinPartitions(
  new CustomHBasePartitioner(100)
)
```

### Advanced Operations

#### 1. Aggregations with Optimization

```scala
def optimizedAggregation(hbaseRDD: RDD[Result]): RDD[(String, Double)] = {
  hbaseRDD
    .mapPartitions { iterator =>
      // Pre-aggregate within partition
      val localAggs = new HashMap[String, Double]()
      
      iterator.foreach { result =>
        val key = extractKey(result)
        val value = extractValue(result)
        localAggs.merge(key, value, _ + _)
      }
      
      localAggs.iterator
    }
    .reduceByKey(_ + _)  // Final aggregation across partitions
}
```

#### 2. Join Operations

Particularly important for HBase data:

```scala
def efficientJoin[K: ClassTag](
  leftRDD: RDD[(K, Result)],
  rightRDD: RDD[(K, Result)]
): RDD[(K, (Result, Result))] = {
  
  // Broadcast smaller RDD if possible
  if (leftRDD.count() < rightRDD.count() * 0.1) {
    val broadcastLeft = leftRDD.sparkContext.broadcast(
      leftRDD.collect().toMap
    )
    
    rightRDD.mapPartitions { iterator =>
      val leftMap = broadcastLeft.value
      iterator.flatMap { case (k, rightValue) =>
        leftMap.get(k).map(leftValue => 
          (k, (leftValue, rightValue))
        )
      }
    }
  } else {
    // Regular join if sizes are comparable
    leftRDD.join(rightRDD)
  }
}
```

## 2.3 Performance Considerations for RDD Operations

### Memory Management

When working with RDD operations, memory management is crucial:

1. Persistence Levels:
```scala
// Choose appropriate storage level
rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
```

2. Partition Sizing:
```scala
// Estimate partition size
def estimatePartitionSize(result: Result): Long = {
  result.getNoOfColumns * 
  result.getFamilyMap(result.getFamilies()(0)).size() *
  8  // Approximate bytes per cell
}
```

[Continued in next section...]