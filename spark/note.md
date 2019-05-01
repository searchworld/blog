# dependency 
## narrow or wide
https://github.com/rohgar/scala-spark-4/wiki/Wide-vs-Narrow-Dependencies

### Transformations with (usually) Narrow dependencies:
```
map
mapValues
flatMap
filter
mapPartitions
mapPartitionsWithIndex
```

### Transformations with (usually) Wide dependencies: (might cause a shuffle)
```
cogroup
groupWith
join
leftOuterJoin
rightOuterJoin
groupByKey
reduceByKey
combineByKey
distinct
intersection
repartition
coalesce
```

## join always shuffle？
if has the same partitioner then not.
http://amithora.com/understanding-co-partitions-and-co-grouping-in-spark/
https://stackoverflow.com/questions/28395376/does-a-join-of-co-partitioned-rdds-cause-a-shuffle-in-apache-spark

# closure
https://spark.apache.org/docs/2.3.3/rdd-programming-guide.html#understanding-closures-

# Accumulators
For accumulator updates performed inside actions only, Spark guarantees that each task’s update to the accumulator will only be applied once, i.e. restarted tasks will not update the value. In transformations, users should be aware of that each task’s update may be applied more than once if tasks or job stages are re-executed.
https://spark.apache.org/docs/2.3.3/rdd-programming-guide.html#accumulators
