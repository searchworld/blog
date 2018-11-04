# Overview
Flink为统一的stream和batch处理提供了两个relational api: Table API & SQL。前者是为Scala和Java集成的查询api，后者是基于实现了SQL标准的Apache Calcite。这两种api可以和DataStream/DataSet进行集成。

# Concepts & Common API
api核心是`Table`，表示一个查询的输入和输出。

## Structure of Table API and SQL Programs
```
// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// create a TableEnvironment
// for batch programs use BatchTableEnvironment instead of StreamTableEnvironment
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register a Table
tableEnv.registerTable("table1", ...)            // or
tableEnv.registerTableSource("table2", ...);     // or
tableEnv.registerExternalCatalog("extCat", ...);

// create a Table from a Table API query
Table tapiResult = tableEnv.scan("table1").select(...);
// create a Table from a SQL query
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...);

// execute
env.execute();
```

## Translate and Execute a Query
Table API & SQL查询会被转成DataStream或者DataSet程序。一个查询用一个logical query plan表示，会转成两个阶段：
- logical plan的优化
- 转成一个DataStream或者DataSet程序

转换的时机：
- Table被发出到TableSink中，比如调用`Table.writeToSink() or Table.insertInto()`
- 指定一个SQL update query，比如调用`TableEnvironment.sqlUpdate() `
- Table被转成一个DataStream或者DataSet

转换之后就可以像一个DataStream或者DataSet程序正常执行了。

## Integration with DataStream and DataSet API

### Register a DataStream or DataSet as Table
```
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as Table "myTable" with fields "f0", "f1"
tableEnv.registerDataStream("myTable", stream);

// register the DataStream as table "myTable2" with fields "myLong", "myString"
tableEnv.registerDataStream("myTable2", stream, "myLong, myString");
```

### Convert a DataStream or DataSet into a Table
```
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, "myLong, myString");
```

### Convert a Table into a DataStream or DataSet
```
// get StreamTableEnvironment. 
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into an append DataStream of Row by specifying the class
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer> 
//   via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataStream<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream<Tuple2<Boolean, X>>. 
//   The boolean field indicates the type of the change. 
//   True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream = 
  tableEnv.toRetractStream(table, Row.class);
```
## Mapping of Data Types to Table Schema
### Position-based Mapping
```
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field names "myLong" and "myInt"
Table table = tableEnv.fromDataStream(stream, "myLong, myInt");
```

### Name-based Mapping
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "f1" only
Table table = tableEnv.fromDataStream(stream, "f1");

// convert DataStream into Table with swapped fields
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
Table table = tableEnv.fromDataStream(stream, "f1 as myInt, f0 as myLong");
```

## Query Optimization
Flink使用 Apache Calcite对查询进行优化和转换，目前的优化包括`projection and filter push-down, subquery decorrelation, and other kinds of query rewriting`，还没有对join的顺序进行优化。
### Explaining a Table
通过`TableEnvironment.explain(table)`可以查看三个计划：
- 关系查询的`Abstract Syntax Tree`，即未优化的计划
- 优化后的logical query plan
- physical query plan