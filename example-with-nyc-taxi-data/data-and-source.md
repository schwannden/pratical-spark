# The Data

The complete NYC taxi records can be found [here](http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml), and specifically, we will be using the [yellow record for 2016/01](https://s3.amazonaws.com/nyc-tlc/trip+data/yellow_tripdata_2016-01.csv), which sized about 1.6G. The data format specification is listed [here](http://www.nyc.gov/html/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf). Put the data file under project directory  `data/taxi201601.csv`

The source is found [here](https://github.com/schwannden/taxi-spark). I used sbt as the build tool for my project. A simple sbt tutorial can be found [here](https://github.com/shekhargulati/52-technologies-in-2016/tree/master/02-sbt).

To compile the project, inside project directory, type `sbt package`

```bash
>  sbt package
[info] Loading project definition from /home/schwannden/spark/taxi-spark/project
[info] Set current project to taxi (in build file:/home/schwannden/spark/taxi-spark/)
[info] Compiling 1 Scala source to /home/schwannden/spark/taxi-spark/target/scala-2.11/classes...
[info] Packaging /home/schwannden/spark/taxi-spark/target/scala-2.11/taxi_2.11-1.0.jar ...
[info] Done packaging.
[success] Total time: 2 s, completed Mar 20, 2017 11:48:12 AM
```

And you can copy the path of the target jar, and start spark-shell with this jar

```Bash
spark-shell --master spark://10.0.0.2:7077 --jars target/scala-2.11/taxi_2.11-1.0.jar
```



