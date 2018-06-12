Data parallel to distributed parallel

shared memory case - one node. Some dataset on one computer.

Shared memory data parallelism:
		- split the data
		- threads independently operate on the data shards in parallel
		- combine when done

 scala parallel collections is a collections abstraction over shared memory and a parallel execution

 this can be expended on distributed datta parallelism

 only difference that there no threads, but different nodes

 we should worry about latency between nodes

 difference:
 	- shared memory: data parallel programming model, data partitioned in memory and operated upon in parallelism
	- distributed: data parallel programming mode. datta partitioned between machines, network in between, operated upon in parallel.

spark implements a distributed data parallel model called resilient distributed datasets

we chunk big data somehow, distribute it over cluster machines and run like it is usual collection


# latency
data parallelism in a distributed setting
distributed collections abstraction from apache spark as an implementation of this paradigm
when distributed, we should worry about:
	- partial failure: crash failures of a subset of the machines involved in a distributed computation
	- latency: certain operations have a much higher latency than other operations due to network communication
	- latency connot be masked completely; it will be an important aspect that also impacts the programming model

important latency numbers:
 //insert picture
1 mb from memory is 100x cheaper than reading 1mb from disk (sequentially)
memory: fstest
disk: slow
network: slowest

Hadoop: batch data processing framework, opensource implementation of google's mapreduce

mapreduce: simple map and mapreduce
			fault tolerance, allows to scale easily from 100 to 1000 nodes. it is important, because node failing is very high.
			So ability to recover from node failure allow:
				- computations on unthinkably large data sets to succeed to completion


why not just hadoop, but spark?:
	- between each step hadoop shuffles datta and write intermediate data to disk, in order to recover from potential failures
	- spark uses different strategy for handling latency and retains fault-tolerance
	- spark main idea to handle latency is: keep all data immutable and in-memory. all operations on data are just functional transformations like regular scala collections. fault tolerance is achieved by reading data somewhere else and replaying functional transformations over original dataset.
	- as a result spark has shown to be 100x more performant than hadoop, while adding even more expressive APIs
	- hadoop operations involve disk and network operations
	- spark operations involve memory and network operations, more operations in memory. it agrresively minimizes its network traffic.
	- spark brings real productivity

# Spark RDDs
RDDs are a lot like immutable sequential or parallel scala collections, like a List.
RDD API:
	- map
	- flatmap
	- filter
	- reduce

most operations on rdds are higher-order functions. they take a function as an argument and typically return RDD

Combinators on RDDs:
	- map
	- flatmap
	- filter
	- reduce
	- fold
	- agregate

RDDs can be created in two ways:
	- transforming an existing RDD:
		- like a call to map on a list.
	- from a SparkContext or SparkSession object:
		- can be thought of as handle to spark cluster. It represents the connection betweeen the spark cluster and running application. It defines a handful of methods which can be used to create and populate a new RDD.
			- parallelize: convert a local Scala collection to an RDD
			- textFile: read a text file from HDFS or a local file system and return an RDD of  string. Mostly used in real world tasks.


 RDDs transformations and actions
	- Transformers: return new collections as results (not single values). eg map, filter, flatmap, groupby
	- Accessors: return single values as results (not collections.) eg reduce, fold, agregate.

So Spark defines transformations and actions on rdds.
Transformations return new rdds as results. they are lazy, their result rdd is not immediately computed
Actions compute a result based on an rdd and either returned or saved to an external storage system (eg HDFS). they are eager, their result is immediately computed.
	Laziness/eagerness is how we can limit network communication using the programming model.

if we run map on rdd, like wordsRdd.map(_length) we should add an action:
lengthsRdd.reduce(_ + _)

common transformations: map, flatmap, filter, distinct. THEY ARE LAZY.
common actions: collect, count, take, reduce, foreach. they are eager.
collect usually used when some operations to reduce dataset are done, it returns all elements of rdd.

benefits of laziness: spark computes rdds the first time they are used in an action. this helps when processing large amounts of data.
eg firstLogsWithErrors = lastYearsLogs.filter(_contains("ERROR")).take(10)
The execution of filter is deferred until the take action is applied. Spark leverages this by analyzing and optimizing the chain of operations before executing it. Spark will not compute intermediate RDDS, instead, it will stop as soon as 10 elements of filtered RDD have been computed. At this point spark stops working.

Why spark is unlike scala collections?
	why spark is good for datascience:
	iteration in hadoop:
		- file system read
		- iter 1
		- file system write
		- file system read
		- iter 2
		- file system write
		- ...
spark can avoid 90% hadoops io
	iter in spark:
		- input
		- iter 1
		- iter 2
		- ...

iterative algorithms
	by default RDDs are recomputed each time you run an action on them. this can be expensive if you need to use a dataset more then once. Spark allows you to control what is cached in memory. To cache an RDD in spark simply call persist() or cache() on it.
There are many ways to configure how your data is persisted.
possible to persist data set:
	- in memory as regular java objects
	- on disk as regular java objects
	- in memory as serialized java objects
	- on disk as serialized java objects
	- both in memory and on disk.
cache: shorthand for using the default storage level wich is in memory only as regular java objects.
persist: persistence can be customized with this method. pass the storage level youd like as a parameter to persist.

//picture of storage levels

Despite similar-looking api to scala collections the deferred semantics of spark rdds are very unlike scala collections.
due to:
	- the lazy semantics of rdd transformation operations
	- users implicit reflex to assume collections are eagerly evaluated
one of the most common performance bottlenecks of newcomers to spark arises from unknowingly re-evaluating several transformations when caching could be used.

#cluster topology matters!
How spark jobs are executed:
spark is organized in a master worker topology (one master, lot of workers)
master is program driver and workers are worker nodes. driver program has spark context in it, workers have executor inside. driver can create new rdds, populate new rdds, etc. you are interacting with driver program, nodes are executing the jobs.
how they communicate:
	they communicate via a cluster manager. it allocates resourcess across cluster, manages scheduling, eg YARN/Mesos.
a spark application is a set of processes running on a cluster. all these processes are coordinated by the driver program.
The driver is:
	- the process where the main() method of your program runs
	- the process running the code that creates a sparkcontext, creates rdds and stages up or sends off transformations and actions.

these processes that run computations and store data for your application are executors. Executors:
	- run the tasks that represent the application
	- return computed results to the driver
	- provide in-memory storage for cached rdds.


Execution of a sprk program:
- the driver program runs the spark app which creates a SparkContext upon startup
- the SparkContext connects to a cluster manager which allocates resources.
- spark acquires executors on nodes in the cluster, which are processes that run computations and store data for your application
- next driver program sends your app code to the Executors
- finally sparkcontext sends tasks for the executors to run.

THE MORAL:
To make effective use of rdds uou have to understand a little bit about how spark works under the hood.



# Week 2

How reduce distributed?
Reduction operations: recall operations such as fold reduce adn aggregate from Scala seq collections.
Reduction operations walk though a collection and combine neigboring elements of the collection together to produce a single combined result (not another collections)
Many spark actions are reduction operations, but some are not. Eg saving things to a file is not reduction operation.

foldLeft is not parallelizable; foldleft: applies a binary operator to a start value and all elements of this collection from left to right.
foldLeft is not parallelizable because for being able to change the result type from A to B forces us to have execute foldLeft sequentially from left to right.

fold enables us to parallelize things, but it restricts us to always returning the same type. It enables us to parallelize using a single function f by enabling us to build parallelizable reduce trees.

Aggregate: seq operator, combination operator. Aggregate is parallelizable and its possible to change the return type.
//lego visualisation pic.
Agregate lets you still do seq-style folds in chunks which change the result type. Additionally requiring the combop function enables building one of these nice reduce trees that we saw is possible with fold to combine these chunks in parallel.

reudction operations on RDDs:
spark:
fold, reduce, aggregate. Spark doesn't give you the option to use foldLeft/Right. That means that if you need change the return type of reduction operation, only chice is aggregate.
Why not serial foldLeft/Right on spark?
Because doing things serially across a cluster is actually difficult. Lots of synchronization. Doesn't make a sens.

Much of time when working with large-scale data the goal is to project down from larger/more complex data types.

## Pair rdds.
Most common in world of big data processing is operation on data in the form of key-value pairs.
	- Manipulating k-v pairs a key choice in design of MapReduce.

Large datasets are often made up of unfathomably large numbers of complex, nested data records. To be able to work with such datasets, it's often desirable to project down these complex datatypes into key-value pairs.

In spark distributed k-v pairs are "Pair RDDs"
Useful because: Pair RDDs allow you to act on each key in parallel or regroup data across the network.
Whe an RDD is created with a pair as its element type, Spark automatically adds a number of extra useful additional methods for such pairs.
Some of the most important extension methods for RDDS containing pairs are:
	- group by key
	- reducreByKey
	- join

Creating a pair rdd:
Pair rdds are most often created from already-existing non-pair RDDs, for example, by using the map operation on RDDs:
rdd: RDD[WikipediaPage]
pairRdd: RDD[(String, String)] = rdd.map(page => (page.title, page.text))

Once created, you can now use transformations specific to key-value pairs such as reduce by key, group by key, join.

## Transformations and Actions on pair RDDs.
Two categories again: transformations and actions.
some important operations on pair rdd (but not avaliable on regular rdds):
	- Transformations:
		- groupByKey
		- reduceByKey
		- mapValues
		- keys
		- join
		- leftOuterJoin/rightOuterJoin
	- Actions:
		- countByKey

groupBy: breaks up a collection into two or more collections according to a function that you pass to it. Result of the function is the key, the collection of results that return key when the function is applied to it. Retuns a Map mapping computed keys to collections of corresponding values.

groupByKey groups all values that have the same key and therefore takes no argument.

reduceByKey can be thought of as a combination of groupByKey and reduce-ing on all the values per key. It's more efficient though, than using each separately.
a little note: reduceByKey takes a function which only cares about the values of the paired RDD. We do not do anything on keys in this function, we only operate on values. This is because we assume that values already somehow grouped by key and now we apply this function to reduce over those values that are in a collection.

mapValues: simply applies a function to only the values in a pairRDD;

countByKey simply counts the number of elements per key in a pair RDD, returning a normal Scala Map (because its an action) mapping from keys to counts.

keys: return an RDD with the keys of each tuple. This method is a transformation and thus returns an RDD because the number of keys in a Pair RDD may be unbounded. It's possible for every value to have a unique key, and thus is may not be possible to collect all keys at one node.

##Joins:
Joins are another sort of transformation on Pair RDDs. They're used to combine multiple datasets. They are one of the most commonly-used operations on Pair RDDs!
There are two kinds of joins:
	- Inner Joins (join)
	- Outer joins (leftOuterJoin/rightOuterJoin)

The key difference between the two is what happens to the keys when both RDDs don't contain the same key.

 Inner joins return a new RDD containing combined pairs whose keys are present in both input RDDs. For inner joins order of join doesn't matter.

 Outer joins return a new RDD containing combined pairs whose keys don't have to be present in both input RDDs. Outer joins are particularly useful for customizing how the resulting joined RDD deals with missing keys. With outer joins, we can decide which RDDs keys are most essential to keep-the left, or the right RDDin the join expression.


# WEEK 3
Shuffling:
To do distributed groupByKey we typically move data between nodes so the data can be collected. This is called shuffling. Shuffles happen and they happen transparently, as a part of operations like groupByKey. It may be enormous hit to performance because it means that spark has to move a lot of data from one node to another and LATENCY happens. To avoid this sometimes use reduceByKey.
important: reduceByKey reduces on the mapper side first. It helps to avoid shuffling
Benefits: by reducing the dataset first, the amount of data sent over the network during the shuffle is greatly reduced. This can result in non-trivial gains in performance.
By default spark uses hash partitioning to determine which key-value pair should be sent to which machine.

Partitioning:
the data within an RDD is split into several partitions.
properties of partitions:
- Partitions never span multile machines, ie tuples in the same partition are guaranteed to be on the same machine.
- Each machine in the cluster contains one or more partitions
- The number of partitions to use is configurable. by default, it equals the total number of cores on all executor nodes.

Two kinds of partitioning available in Spark:
- Hash Partitioning
- Range partitioning

Customizing a partitioning is only possible on Pair RDDs
Hash partitioning:
attempts to spread data evenly across partitions based on the key.
group by key first computes per tuple (k, v) its partition p:
p = k.hashCode() % numPartitions.
Range partitioning:
Pair RDDs may contain keys that have and ordering defined: Itn, Char, String, ....
For such RDDs, range partitioning may be more efficient. Using a range partitioner, keys are partitioned according to:
- an ordering for keys
- a set of sorted ranges of keys.
- tuples with keys in the same range appear on the same machine.

How to set partitioning for our data:
- Call partitionBy on an RDD, providing an explicit Partitioner
- Using transformations that return RDDs with specific partitioners.

Creating a Range parittioner requires:
- Specifying the desired number of partitions
- providing a pair RDD with ordered keys. This RDD is sampled to create a suitable set of sorted ranges.

Important: the result of partitionBy should be persisted. Otherwise, the partitioning is repeatedly applied (involving shuffling) each time the partitioned RDD is used.

	Partitioner from parent RDD:
Pair RDDS that are the result of a transformation on a partitioned Pair RDD typically is configured to use the hash partitioner that was used to construct it.
Automatically-set partitioners:
Some operations on RDDs automatically result in an RDD with a known partitioner - for when it makes sense.
For exmaple, by default, when using sortByKey, a RangePartitioner is used. Further, the default partitioner when using groupByKey, is a HashPartitiioner, as we saw earlier.

Operations on Pair RDDs taht hold to (and propagate) a partitioner:
- cogroup
- groupWith
- join
- left/rightOuterJoin
- group/reduceByKey
- fold/combineByKey
- partitionBy
- sort
- mapValues,flatMapValues,filter (if parent has a partitioner)

All other operations will produce a result without a partitioner.
If use map or flatMap on a partitioned RDD that RDD will lose its partitioner.

Why operations producing a result without a partitioner?
Consider the map transformation. Give that we have a hash partitioned Pair RDD, why would it make sense for map to lose the partitioner in its result RDD? Because its possible for map to change the key.

Optimizing with Partitioners.
Sometimes is useful to just use partitionBy in the beginning and then perstist it.

How do I know a shuffle will occur?
A shuffle can occur when the resulting RDD depends on other elements from the same RDD or another RDD.
It also can be figured out whether a shuffle has been planned/executed via:
- The return type of certain trasformations
- using function toDebugString to see its execution plan.

operations that might cause a shuffle:
- cogroup
- groupWith
- all sorts of joins
- all sorts of "ByKey" operations
- distinct
- intersection
- repartition
- coalesce

There are still a few ways to use these operations and avoid shuffle. Examples:
- reduceByKey running on a pre-partitioned RDD will cause the values to be computed locally, requiring only the final reduced value has to be sent from the worker to the driver
- join called on two RDDs that are pre-partitioned with the same parititoner and cached on the same machine will cause the join to be computed locally with no shuffling across the network.

Morale: data organisation on the cluser and operations executed on it MATTERS!

##Wide vs Narrow dependencies
Not all trasformations are equal
Some transformations significantly more expensive in term of latency than others. E.g. requiring lots of data to be transferred over the network, sometimes unnecessarily.
Computations on RDDs are represented as a lineage graph; a directed acyclic graph representing the computations done on the RDD.

RDDs are made up of 2 imporatnt parts (but are made up of 4 parts in total). RDDs are represented as:
- Partitions. Atomic pieces of the dataset. One or many per compute node.
- Dependencies. Models relationship between this RDD and its partitions with the RDD(s) it was derived from.
- A function for computing the dataset based on its parent RDDs
- Metadata about its partitioning scheme and data placement.

RDD dependencies encode when data must move across the network.
Transformations cause shuffles. Transformations can have two kinds of dependencies:
- narrow dependencies: each partition of the parent RDD is used by at most one partition of the child RDD. They are fast, no shuffle necessary. Optimizations like pipelining possible (when you can group together many transformations into one pass).Transformations: map, filter, union, mapValues, flatMap, mapPartitions, mapPartitionsWithIndex join with co-partitioned input.
- wide dependencies: each partition of the parent RDD may be depended on by multiple child partitions. They are slow, requires all or some data to be shuffled over the network. Transformations: groupByKey, cogroup, all sorts of "byKey", distinct, intersection, repartition, coalesce, groupBy, joins with inputs not co-partitioned

How it can be figured out?
dependencies method on RDDs.
dependencies returns a sequence of Dependency objects, which are actually the dependencies used by Sparks scheduler to know how this RDD depends on other RDDs. The sort of dependency objects the dependencies method may return include:
- Narrow:
	- OneToOne
	- Prune
	- Rande
- Wide:
	- Shuffle
toDebugString prints out a visualiation of the RDDs lineage, and other information pertinent to scheduling. For example, indentations in the output separate groups of narrow transformations that may be pipelined together with wide transformations that reuire shuffles. These groupings are called stages.

Lineages graph are the key to fault tolerance in spark. Ideas from functional programming enable fault tolerance in Spark:
- RDDs are immutable
- we use higher-order functions like map, flatMap, filter to do functional transformations on this immutable data
- A function for computing the dataset based on its parent RDDs also is part of an RDDs representation.

This enables us to keep track of dependency information between partitions as well, and recover from failures by recomputing lost partitions from lineage graphs.
This allows fault tolerance w/out having to checkpoint write data to disk. In-memory + fault-tolerant.
Recomputing missing partitions fast for narrow dependencies, but slow for wide, because for wide it is really tons of recomputing.

# Week 4
## Structure and Optimization
All data isn't equal structurally. unstructured: log files, images; semi-structured: json, xml; structured: db tables.
Spark + regular RDDs don't know anything about the schema of the data it's dealing with. Given an arbitrary RDD, Spark knows that the RDD is parameterized with arbitrary types, but it doesn't know anything about these types structure.
Spark can't see inside objects or analyze how it may be used, and to optimize based on that usage. It's opaque.
Optimization can be done on DB lvl.
Same can be said about computation. In Spark:
- We do functional transformations on data.
- We pass user-defined function literals to higher-order functions like map, flatMap, filter.

Like the data Spark operates on, function literals too are completely opaque to Spark. User can do anything insede one of these.
In a database/Hive:
- We do declarative transformations on data.
- Specialized/structured, pre-defined operations.

There are fixed set of operations, fixed set of types they operate on. Optimizations the norm.

Optimizations + Spark: RDDs operate on unstructured data, and there are few limits on computation; your computations are defined as functions that you've written yourself, on your own data types. But as we saw, we have to do all the optimization work ourselves.
Spark SQL makes optimizations from DB lvl avaliable to us.

## Spark SQL.
Spark SQL makes possible to seamlessly intermix SQL and Scala and to get all of the optimizations we're used to in the databases community on Spark jobs.
It is a spark module for structured data processing. It is implemented as a library on top of Spark.

Spark SQL goals:
- Support relational processing both within Spark programs on RDDs and on external data sources with a friendly API. Sometimes it's more desirable to express a computation in SQL syntax than with functional APIs and vice versa.
- High performance, achieved by using techniques from research in databases
- Easily support new data sources such as semi-structured data and external


Three main APIs:
- SQL literal syntax
- DataFrames
- DataSets

Two specialized backend components:
- Catalyst, query optimizer;
- Tungsten, off-heap serializer. It is very efficient representation of Scala objects off-heap away from the garbage collector.

Everything about SQL is structured.
relation is just a table
Attributes are colums. Rows are records or tuples.

Spark SQL core abstraction is DataFrame. It is conceptually equivalent to a table in a relational database. Conceptually, they are RDDs full of records with a known schema.
DataFrames are untyped. Scala compiler doesn't check the types in its schema. DataFrames contain Rows which can contain any schema.
Transformations on dataframes are also known as untyped transformations.

To get started using Spark SQL, everything starts with the SparkSession.

The SparkSession is basically the SparkContext for everything in Spark SQL. Usage is simple: import and use builder.

DataFrames created in two ways:
- From an existing RDD. Either with schema inference, or with an explicit schema.
	- via method toDF(column names). If with no params, Sparl will assign numbers as attributes to DataFrame. If you already have an RDD containing some kind of case class instance, Spark can infer the attributes from the case class's fields.
	- with explicitly specified schema. Three steps:
		- Create an RDD of Rows from the original RDD
		- Create the schema represented by a StructTyp (StructField for fields, StructType)e matching the structure of Rows in the RDD created in Step 1
		- Apply the schema to the RDD of rows via createDataFrame method provided by SparkSession.
- Reading in specific data source from file. COmmon structured or semi-structured formats such as JSON. via read(path) method. Sources for automatically creating data frames: json, csv, parquet, jdbc.


Once created, sql-familiar syntax can be used to operated. We just have to register DataFrame as a temporary SQL view first.


DataFrames API:
DataFrames operate on a restricted set of data types in order to enable optimization opportunities.
Primitieves, String, Timestapms, Arrays, Maps, case classes.
In order to access any of these data types, basic or complex, you must first import Sqprk SQL types.
DataFrames API quite similar to SQL. Example methods: select, where, limit, orderBy, groupBy, join.
printSchema prints the schema of DataFrame in a tree format.

Selection and working with columns can we do in three ways:
1. Using $-notation: df.filter($"age" > 18)
2. Referrring to the Dataframe: df.filter(df("age") > 18)
3. Using SQL query string: df.filter("age > 18 ")

Cleaning data iwth DataFrames:
Unwanted data can be dropped using drop() method.
Replacing unwanted values:
1. fill() replaces all occurrences of null or NaN in numeric columns with specified value and returns a new DataFrame.
2. fill(Map("minBalance" -> 0)) replaces all occurrences of null or NaN in specified column with specified value and returns a new DataFrame
3. replace(Array("id"), Map(1234 -> 8923)) replaces specified value in a pecified column with specified replcement value and returns a new DataFrame.

Actions on DataFrames:
collect, count, first, show (displays the top 20 rows of DataFrame in a tabular form), take.

Joins on DataFrames:
Joins on DataFrames are pretty similar to joins on Pair RDDs, with the one major difference that in DataFrames we have to specify which columns we should join on.
Several types of joins: inner, outer, left_outer, right_outer, leftsemi.

Pros of using spark SQL: The catalyst optimizer:
It allows:
- reordering operations
- reduce the amount of data we must read
- purne unneeded partitioning

Tungsten -- spark sql off-heap data encoder
It provides:
- highly-specialized data encoders  -- it has schema information and tightly pack serialized data into memory. Therefore, more data can fit in memory and faster serialization/deserialization.
- column-based
- off-heap (free from garbage collection overhead) -- regions of memory off the heap, manually managed by tungsten, so as to avoid garbage collection overhead and pauses.

Limitations of dataframes:
- untyped: code compiles, but you get runtime exceptions when you attempt to run a query on a column that doesn't exist. Would be nice if this was caught at compile time like were used to.
- limited data types: if your data can't be expressed by case classes/products and standart spark sql data types, it may be difficult to ensure that a Tungsten encoder exists for your data type.
- requires semi-structured/structured data: if your unstructured data cannot be reformulated to adhere to some kind of schema, it would be better to use RDDs

## Datasets
DataFrames are actually datasets, parametrized with Row.
 Dataset:
 - Can be thought of as typed distributed collections of data
 - Dataset api unifies the dataframe and rdd apis.
 - datasets require structured/semi-structured data. schemas and encoders core part of datasets.
 - It is like a compromise between RDDs and dataframes. More information than on dataframes and more optimizations than on rdds.
 - you can still use relational dataframe operations.
 - also there are more typed operations that can be used as well.
 - higher order functions like ap, flatmap.
 - almost type-safe API
- typed and untyped transformations.


Aggregators[-in, BUF, OUT]: a class that helps generically aggregate data.
- IN: input type to the aggreagtor.
- BUF: intermediate type during aggregation
- OUT: type of the output.

Encoders:
Encoders are what convert data between JVM objects and Spark SQL specialized internal tabular representation.
Encoders are highly specialized, optimized code generators that generate custom bytecode for serialization and deserialization of your data.
The serialized data is stored using Tungsten binary format, allowing for operations on serialized data and improved memory utilization.

Why Kryo/Java serialzation are not good enough?

What sets them apart from regular Java or Kryo serialization:
- Limited to and optimal for pirmitives and case classes, Spark SQL data types, which are well-understood.
- They contain schema information, which makes these highly optimized code generators possible, and enables optimization based on the shape of the data. Since Spark understands the structure of data in Datasets, it can create a more optimal layout in memory when caching Datasets.
- Using less memory than Kryo/Java serialization.
- 10x fater than Kryo serialization.

Two ways to create encoders:
- Automatically: via implicits from a SparkSession
- Explicitly via org.apache.spark.sql.Encoder

Some examples of Encoder creation methods in Encoders:
- INT/LONG/STRING for nullable primitives
- scalaInt/scalaLong/... for scala primitives
- product/tuple for scala product and tuple types.

 When to use Datasets:
 - you have structured/semi-structured data
 - you want typesafety
 - you need to work with functional apis
 - you need good performance, but it doesn't have to be the best

When to use Dataframes:
- you have structured/semi-structured data
- you want the best possible performance, automatically optimized for you

When to use RDDs:
- you have unstructured data
- you need to fine-tune and maange low-level details of RDD computations
- you have complex data types that cannot be serialized with Encoders.

Catalyst can't optimieze all operations.
- when using datasets with higher-order functions like map, you miss out on many Catalyst optimizations.
- When using datasets with relational operations like select, you get all of catalyst optimizations
- though not all operations on datasets benefit from catalysts optimizations, tungsten is still always running under the hood of datasets, storing and organizing data in a highly optimized way, which can result in large speedups over RDDs

Limited data types:
if your data can't be expressed by case classes and standart spark sql data types, it may be difficult to ensure that a tungsten encoder exists for your data type.
Structured/semi-structured data.
