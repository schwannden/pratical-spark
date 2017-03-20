# Interactive Shell

An interactive shell is extremely important for data analysis because you want to do analysis and see results quickly instead of compiling your code each time and wait for the result to come back. Now that we have a cluster running, start spark-shell and bind it to this cluster

```bash
spark-shell --master spark://10.0.0.2:7077
```

Now you should see in Web UI that an application appears

![](/assets/6.png)Run try some scala codes

```Scala
scala> 1
res0: Int = 1

scala> def factorial(n: Int) = if (n == 0) 1 else n * factorial(n-1)
factorial: (n: Int)Int

scala> factorial(10)
res1: Int = 3628800
```

None of this code involves Spark API, so let's try something with spark. Copy the following codes to your spark console without worrying too much about what it means.

```Scala
import java.util.Random
import org.apache.spark.mllib.linalg.{Vector, Vectors}
import org.apache.spark.mllib.clustering.{KMeans, KMeansModel}
val points = (1 to 1000000).par.map(i => {val r = new Random; Vectors.dense(r.nextDouble, r.nextDouble)})
val pointsRDD = sc.parallelize(points.toVector)
val numClusters = 2
val numIterations = 20
val clusters = KMeans.train(pointsRDD, numClusters, numIterations)
```

In this code snippet we create one million 2 dimensional points, and transform it to spark's RDD data structure \(RDD\). We then run K means clustering on these large dataset. When the job is running, go to the application Web UI \(click the blue "spark shell"\), or if you followed my step, the url is 10.0.0.2:4040, you should see jobs completed and undergoing

![](/assets/7.png)

In my setup, everything finished in 10 seconds. If it take too long for you, try to make your datasize smaller.



