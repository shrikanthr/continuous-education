# Chapter 3: Memory Management and Performance Optimization

## 3.1 Spark Memory Architecture

### Memory Pools and Management

Spark's memory management is divided into several distinct regions, each serving specific purposes and managed differently:

#### 1. JVM Heap Structure
```
Total Heap Memory
|
+-- Reserved Memory (300MB)
|
+-- User Memory (300MB)
|
+-- Spark Memory
    |
    +-- Storage Memory
    |   |
    |   +-- RDD Cache
    |   +-- Broadcast Variables
    |   +-- Unroll Memory
    |
    +-- Execution Memory
        |
        +-- Shuffles
        +-- Joins
        +-- Aggregations
```

The actual memory allocation is controlled by several key parameters:

```scala
// Total memory available for storage and execution
spark.memory.fraction = 0.75  // Default
spark.memory.storageFraction = 0.5  // Default

// Calculation
val availableMemory = (JVM_HEAP - RESERVED_MEMORY) * spark.memory.fraction
val storageMemory = availableMemory * spark.memory.storageFraction
val executionMemory = availableMemory * (1 - spark.memory.storageFraction)
```

### Dynamic Memory Management

Spark implements a dynamic borrowing mechanism between storage and execution memory:

```scala
class MemoryManager {
  private var storageMemoryUsed: Long = 0
  private var executionMemoryUsed: Long = 0
  
  def acquireExecutionMemory(required: Long): Long = {
    synchronized {
      if (required <= availableExecutionMemory) {
        executionMemoryUsed += required
        required
      } else {
        // Can evict storage memory if needed
        val available = Math.max(
          availableExecutionMemory,
          availableExecutionMemory + availableStorageMemoryToEvict
        )
        Math.min(required, available)
      }
    }
  }
}
```

## 3.2 Memory Optimization Techniques

### 1. RDD Persistence Strategies

Different storage levels have various memory implications:

```scala
// Memory only - fastest but can cause OOM
rdd.persist(StorageLevel.MEMORY_ONLY)

// Memory and disk - safe but slower
rdd.persist(StorageLevel.MEMORY_AND_DISK)

// Serialized memory - more space efficient
rdd.persist(StorageLevel.MEMORY_ONLY_SER)

def chooseStorageLevel(
  rdd: RDD[_],
  estimatedSize: Long,
  availableMemory: Long
): StorageLevel = {
  if (estimatedSize * 1.5 < availableMemory) {
    // Safe to keep in memory
    StorageLevel.MEMORY_ONLY
  } else if (estimatedSize < availableMemory) {
    // Use serialization to save space
    StorageLevel.MEMORY_ONLY_SER
  } else {
    // Fall back to disk
    StorageLevel.MEMORY_AND_DISK_SER
  }
}
```

### 2. Partition Size Optimization

Proper partition sizing is crucial for memory management:

```scala
def optimizePartitions[T: ClassTag](
  rdd: RDD[T],
  targetSize: Long = 128 * 1024 * 1024  // 128MB default
): RDD[T] = {
  
  // Calculate current partition sizes
  val partitionSizes = rdd.mapPartitionsWithIndex { (idx, iter) =>
    Iterator.single((idx, iter.size))
  }.collect()
  
  // Calculate optimal partition count
  val totalSize = partitionSizes.map(_._2.toLong).sum
  val optimalPartitions = Math.max(
    1,
    Math.ceil(totalSize.toDouble / targetSize).toInt
  )
  
  // Repartition if needed
  if (Math.abs(optimalPartitions - rdd.getNumPartitions) > 
      rdd.getNumPartitions * 0.2) {  // 20% threshold
    rdd.coalesce(optimalPartitions, shuffle = true)
  } else {
    rdd
  }
}
```

## 3.3 Memory-Aware Processing Patterns

### 1. Incremental Processing

When dealing with large HBase snapshots:

```scala
def processLargeSnapshot(
  snapshot: HBaseSnapshotRDD,
  batchSize: Int = 10000
): RDD[ProcessedResult] = {
  
  // Process in batches within each partition
  snapshot.mapPartitions { iterator =>
    iterator.grouped(batchSize).flatMap { batch =>
      // Process batch
      val results = new ArrayBuffer[ProcessedResult]()
      
      batch.foreach { result =>
        // Process each result
        val processed = processResult(result)
        results += processed
        
        // Optional: Clear result to help GC
        result.clear()
      }
      
      results
    }
  }
}
```

### 2. Memory-Efficient Aggregations

For large-scale aggregations:

```scala
def memoryEfficientAggregation[K: ClassTag, V: ClassTag](
  rdd: RDD[(K, V)],
  windowSize: Int = 1000000
): RDD[(K, V)] = {
  
  // Two-phase aggregation
  val preAggregated = rdd.mapPartitions { iterator =>
    iterator
      .grouped(windowSize)
      .map { window =>
        // Aggregate within window
        window.groupBy(_._1)
          .mapValues(values => 
            values.map(_._2).reduce(combineValues)
          )
          .iterator
      }
      .flatten
  }
  
  // Final aggregation
  preAggregated.reduceByKey(combineValues)
}
```

## 3.4 Performance Monitoring and Tuning

### 1. Memory Usage Monitoring

Implementing custom memory monitors:

```scala
class MemoryMonitor(
  sc: SparkContext,
  intervalMs: Long = 5000  // 5 seconds
) extends Serializable {
  
  private val accumulator = sc.longAccumulator("memoryUsage")
  
  def monitor[T](rdd: RDD[T]): RDD[T] = {
    rdd.mapPartitions { iterator =>
      // Start monitoring thread
      val monitor = new Thread(() => {
        while (!Thread.currentThread.isInterrupted) {
          val usage = Runtime.getRuntime.totalMemory -
                     Runtime.getRuntime.freeMemory
          accumulator.add(usage)
          Thread.sleep(intervalMs)
        }
      })
      
      monitor.start()
      
      // Process partition
      val result = iterator.toArray
      
      // Stop monitoring
      monitor.interrupt()
      
      result.iterator
    }
  }
}
```

### 2. Performance Metrics Collection

```scala
class PerformanceMetrics(sc: SparkContext) {
  private val processTimeAcc = sc.longAccumulator("processTime")
  private val recordCountAcc = sc.longAccumulator("recordCount")
  private val memorySpikeAcc = sc.longAccumulator("memorySpikes")
  
  def track[T](rdd: RDD[T]): RDD[T] = {
    rdd.mapPartitions { iterator =>
      val startTime = System.nanoTime()
      var count = 0L
      
      val result = iterator.map { record =>
        count += 1
        
        // Check memory usage
        val memoryUsage = Runtime.getRuntime.totalMemory -
                         Runtime.getRuntime.freeMemory
        if (memoryUsage > threshold) {
          memorySpikeAcc.add(1)
        }
        
        record
      }
      
      processTimeAcc.add(System.nanoTime() - startTime)
      recordCountAcc.add(count)
      
      result
    }
  }
}
```

### 3. Adaptive Optimization

Implementing dynamic optimization based on runtime metrics:

```scala
class AdaptiveOptimizer(sc: SparkContext) {
  private val metrics = new PerformanceMetrics(sc)
  
  def optimize[T: ClassTag](rdd: RDD[T]): RDD[T] = {
    // First pass with monitoring
    val monitoredRDD = metrics.track(rdd).cache()
    monitoredRDD.count()  // Force evaluation
    
    // Analyze metrics and adjust
    val processTime = metrics.getProcessTime
    val recordCount = metrics.getRecordCount
    val memorySpikes = metrics.getMemorySpikes
    
    if (memorySpikes > threshold) {
      // Switch to more memory-efficient processing
      optimizeForMemory(monitoredRDD)
    } else if (processTime / recordCount > timeThreshold) {
      // Switch to more CPU-efficient processing
      optimizeForCPU(monitoredRDD)
    } else {
      monitoredRDD
    }
  }
}
```

## 3.5 HBase-Specific Optimizations

### 1. Scan Optimization

```scala
def optimizeScan(scan: Scan, metrics: ScanMetrics): Scan = {
  // Adjust batch size based on row size
  val avgRowSize = metrics.getAvgRowSize
  scan.setBatch(Math.max(100, 1024 * 1024 / avgRowSize))
  
  // Adjust caching based on available memory
  val availableMemory = Runtime.getRuntime.maxMemory * 0.2  // 20%
  scan.setCaching(Math.max(1000, 
    (availableMemory / avgRowSize).toInt))
  
  // Disable block cache for full table scans
  scan.setCacheBlocks(false)
  
  scan
}
```

[Continued in next section...]
