# Spark vs Hadoop

1. Design
   1. Hadoop is more like a distributed data framework, while Spark was built from ground up as a distributed computing framework. What's the difference? A distributed data framework focus on data, so if you want to build algorithm on top of it, you need to use its data API's and translate your algorithm to the language using Hadoop API. 

   2. In spark, they provide abstractions on parallel data structure \(RDD\), so you only need to make sure you do parallelizable computation \(map, filter, reduce, fold, aggregate, etc\) on these spark data structure \(which is almost the same as the native Scala immutable collection\), and the work will be distributed for you internally. You can now focus on the algorithm, instead of the implementation
2. Run Time
   1. For each computation iteration, Hadoop distributes it's computation over multiple nodes, writing results to files, then collecting the data back, then distribute data again for next iteration. The main overhead occur in file read, write, and network delay.
   2. Spark is built on top of Scala, which supports lazy evaluation. So it accumulated all the computations, send it over all the nodes, and computation remains mostly in memory. Data is only collected back to the master machine when all computation is finished. The results in a much faster framework because network overhead are minized and computation occurs mostly in memory.
3. Code Size and Style
   1. Hadoop's code size is huge and mostly not reusable. With its source codes reaching 100000 lines, the database library, graph library, and streaming library still costing 60000~100000 lines of code.
   2. Spark built on top of Scala, which is already an elegant language. Spark-core has about 40000 lines of code, with graph and streaming library costing only about 10000 lines of code. Spark SQL has almost 40000 lines of code too, but mostly are about specific optimizations that make data query much faster.





