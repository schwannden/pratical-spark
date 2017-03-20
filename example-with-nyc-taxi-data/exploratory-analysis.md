### Read the data

```Scala
import preprocess._
import learn._
val file = "data/taxi201601.csv"
val data = new TaxiData(getData(sc, file))
```

### Anomalies

Quickly look at its summary

```Bash
pickHour dropHour pasCount distance      pickLoct         dropLoct         tip
13.546,  13.558,  1.67,    4.648,        -72.819, 40.114, -72.887, 40.153, 0.497
40.855,  41.751,  1.755,   8886929.359,  84.069, 25.512,  79.224, 24.043,  0.002
```

First line is the mean of the feature column, second line is the variance. Notice 2 anomalies:

1. Checking the data format specification, the unit for trip distance is mile, and a variance of 8 million miles is highly unlikely
2. Since the taxi record are in New York, they should have very similar longitude and latitudes, the variance of longitudes and latitudes are too large.

Let's look at anomalies in coordinates first.

```Bash
scala> data.pickLocation.take(10)
res1: Array[org.apache.spark.mllib.linalg.Vector] = Array([-73.99037170410156,40.73469543457031], [-73.98078155517578,40.72991180419922], [-73.98455047607422,40.6795654296875], [-73.99346923828125,40.718990325927734], [-73.96062469482422,40.78133010864258], [-73.98011779785156,40.74304962158203], [-73.99405670166016,40.71998977661133], [-73.97942352294922,40.74461364746094], [-73.94715118408203,40.791046142578125], [-73.99834442138672,40.72389602661133])
```

Looks normal. but I am guessing some of the coordinates are zero due to missing values. Indeed

```Bash
scala> data.pickLocation.filter(v => v(0) == 0 || v(1) == 0).take(10)
res3: Array[org.apache.spark.mllib.linalg.Vector] = Array([0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0], [0.0,0.0])
```

So let's sanitize the data and see how much corrupted data data exists

```Bash
scala> data.pickLocation.count
res4: Long = 10906858
scala> data.sanitize.pickLocation.count
res5: Long = 10720867
```

About 1.8% of the data is corrupted, that's enough to have a huge impact on clustering algorithm. Looking at the summary of sanitized data again, the coordinates' variance are much more reasonable

```Bash
scala> data.sanitize.summary
res6: preprocess.TaxiData#Summary =
pickHour dropHour pasCount distance    pickLoct        dropLoct          tip
13.547,  13.559,  1.674,  4.685,       -73.974, 40.75, -73.974,  40.751, 0.497
40.844,  41.741,  1.765,  9041104.001, 0.006, 0.001,   0.006,  0.002,    0.001
```

### Clustering

Now the coordinates data is clean, run the mixture of Gaussian on pick up locations and drop off locations respectively and see the results \(in my setup, each result takes about 30 seconds to finish\).

```Bash
scala> val res1 = runGmm(data.sanitize.pickLocation)

scala> printGmm(res1)
weight=0.973942
long=-73.97775028258083,lag=-73.97775028258083
sigma=
0.003093193517048202  3.633584970595026E-4  
3.633584970595026E-4  8.639808459437903E-4  

weight=0.026058
long=-73.80042105846606,lag=-73.80042105846606
sigma=
0.0960891637873732     0.0020998083579730036  
0.0020998083579730036  0.014274843331975628
```

Each result takes about 30 seconds to finish. And printing the results, we get

```Bash
scala> val res2 = runGmm(data.sanitize.dropLocation)

scala> printGmm(res2)
weight=0.988347
long=-73.97530655908533,lag=-73.97530655908533
sigma=
0.0036962823296648813  4.194800317387332E-4   
4.194800317387332E-4   0.0013708618468045795  

weight=0.011653
long=-73.81792478925945,lag=-73.81792478925945
sigma=
0.2573087947597205    0.011882345001095702  
0.011882345001095702  0.0491915607547574
```

Both showed large weight on only one cluster. 

### Normalization

Increasing the number of cluster to 10 and see what happened.

```Bash
scala> val res1 = runGmm(data.sanitize.pickLocation, k = 10)
Running Time: 11.3973 minutes                                                   
res1: org.apache.spark.mllib.clustering.GaussianMixtureModel = org.apache.spark.mllib.clustering.GaussianMixtureModel@6ab00916
```

First we note it is taking way too long to! Why? Because coordinates of data points are too similar to each other. Since most coordinates are in New York, most of the data points differ only in the last three digits. And in iteration based algorithm like Mixture of Gaussian, the convergence criteria is based on the error between two iteration. And this high similarity in data will cause convergence to be very slow. Let's normalized it and run again

```Bash
scala> res1 = runGmm(data.sanitize.pickLocation, k = 10, normalized = true)
Running Time: 6.572133333333333 minutes
```

The speed increase by 100%! To print the result, we need to denormalize it

```Bash
scala> import org.apache.spark.mllib.stat.Statistics
scala> val mean = Statistics.colStats(data.sanitize.pickLocation).mean.toArray
scala> printGmm(res1, mean, factor)
weight=0.936235
long=-73.98102833713932,lag=40.74300976084601
sigma=
663.8663793407143  273.111052617864   
273.111052617864   527.5374370679135  

weight=0.063646
long=-73.85736894098412,lag=40.86666915700122
sigma=
10310.617448639501   -2008.5165690037938  
-2008.5165690037938  4544.761368241197    

weight=0.000105
long=-73.85693348955202,lag=40.86710460843332
sigma=
4316281.279067948  352306.8743092276  
352306.8743092276  730761.1109614102  

weight=0.000011
long=-70.37482969461448,lag=44.34920840337087
sigma=
5.947278094258955E7  1337558.4058436945    
1337558.4058436945   1.4887654309033044E7
[omitted further output]
```

### Counting Peak Hour

Then we look at the peak taxi hours. The following function is a simple practice on the transform and map operation on RDD.

```Bash
scala> val pickHour = data.pickHour.map.groupBy(v => v(0)).mapValues(_.size)

scala> pickHour.collect.sortWith((x, y) => x._2 > y._2)
res14: Array[(Double, Int)] = Array((18.0,692093), (19.0,682974), (20.0,624418), (21.0,608035), (17.0,591940), (22.0,581287), (15.0,564002), (14.0,562636), (12.0,535279), (13.0,531803), (16.0,511344), (11.0,502554), (9.0,491927), (8.0,491429), (23.0,481811), (10.0,480614), (7.0,404567), (0.0,396034), (1.0,300325), (6.0,236287), (2.0,229220), (3.0,169130), (4.0,125601), (5.0,111548))

scala> val dropHour = data.dropHour.map.groupBy(v => v(0)).mapValues(_.size)

scala> dropHour.collect.sortWith((x, y) => x._2 > y._2)
res15: Array[(Double, Int)] = Array((19.0,703937), (18.0,686466), (20.0,635415), (21.0,608888), (22.0,590500), (15.0,567494), (17.0,560108), (14.0,548168), (12.0,535878), (13.0,527809), (16.0,522818), (23.0,504585), (9.0,498030), (11.0,493334), (10.0,480787), (8.0,470627), (0.0,416916), (7.0,364426), (1.0,319567), (2.0,243215), (6.0,207235), (3.0,177567), (4.0,137100), (5.0,105988))

```

We see the peak pick up hour is at night 18:00 while the peak drop hour is a little bit later 17:00, which intuitively makes sense.

