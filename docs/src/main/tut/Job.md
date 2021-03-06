# Job\[A\]

All operations on `TypedDataset` are lazy. An operation either returns a new
transformed `TypedDataset` or a `Job[A]`, where `A` is the result of running a
non-lazy computation in Spark. `Job` serves several functions:

- Makes all operations on a `TypedDataset` lazy, which makes them more predictable compared to having
few operations being lazy and other being strict
- Allows the programmer to make expensive blocking operations explicit
- Allows for Spark jobs to be lazily sequenced using monadic composition via for-comprehension
- Provides an obvious place where you can annotate/name your Spark jobs to make it easier
to track different parts of your application in the Spark UI

The toy example showcases the use of for-comprehension to explicitly sequences Spark Jobs.
First we calculate the size of the `TypedDataset` and then we collect to the driver
exactly 20% of its elements:

```tut:invisible
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.SparkSession
import frameless.functions.aggregate._
import frameless.TypedDataset

val conf = new SparkConf().setMaster("local[*]").setAppName("frameless repl").set("spark.ui.enabled", "false")
val spark = SparkSession.builder().config(conf).appName("REPL").getOrCreate()
implicit val sqlContext = spark.sqlContext
spark.sparkContext.setLogLevel("WARN")

import spark.implicits._
```

```tut:book
val ds = TypedDataset.create(1 to 20)

val countAndTakeJob =
  for {
    count <- ds.count()
    sample <- ds.take((count/5).toInt)
  } yield sample

countAndTakeJob.run()
```

The `countAndTakeJob` can either be executed using `run()` (as we show above) or it can
be passed along to other parts of the program to be further composed into more complex sequences
of Spark jobs.

```tut:book
import frameless.Job
def computeMinOfSample(sample: Job[Seq[Int]]): Job[Int] = sample.map(_.min)

val finalJob = computeMinOfSample(countAndTakeJob)
```

Now we can execute this new job by specifying a [group-id][group-id] and a description.
This allows the programmer to see this information on the Spark UI and help track, say,
performance issues.

```tut:book
finalJob.
  withGroupId("samplingJob").
  withDescription("Samples 20% of elements and computes the min").
  run()
```


```tut:invisible
spark.stop()
```

[group-id]: https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext@setJobGroup(groupId:String,description:String,interruptOnCancel:Boolean):Unit
