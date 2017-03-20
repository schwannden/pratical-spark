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
scala> val pickHour = data.pickHour.groupBy(v => v(0)).mapValues(_.size)

scala> pickHour.collect.sortWith((x, y) => x._2 > y._2)
res14: Array[(Double, Int)] = Array((18.0,692093), (19.0,682974), (20.0,624418), (21.0,608035), (17.0,591940), (22.0,581287), (15.0,564002), (14.0,562636), (12.0,535279), (13.0,531803), (16.0,511344), (11.0,502554), (9.0,491927), (8.0,491429), (23.0,481811), (10.0,480614), (7.0,404567), (0.0,396034), (1.0,300325), (6.0,236287), (2.0,229220), (3.0,169130), (4.0,125601), (5.0,111548))

scala> val dropHour = data.dropHour.groupBy(v => v(0)).mapValues(_.size)

scala> dropHour.collect.sortWith((x, y) => x._2 > y._2)
res15: Array[(Double, Int)] = Array((19.0,703937), (18.0,686466), (20.0,635415), (21.0,608888), (22.0,590500), (15.0,567494), (17.0,560108), (14.0,548168), (12.0,535878), (13.0,527809), (16.0,522818), (23.0,504585), (9.0,498030), (11.0,493334), (10.0,480787), (8.0,470627), (0.0,416916), (7.0,364426), (1.0,319567), (2.0,243215), (6.0,207235), (3.0,177567), (4.0,137100), (5.0,105988))
```

We see the peak pick up hour is at night 18:00 while the peak drop hour is a little bit later 17:00, which intuitively makes sense.

### Difference Between Cluster

Suppose we want to know if there is any difference between short and long distance trip, first we need to define "short" and "long." Remember at first we see that the variance of distances is too large, let's see why.

```Bash
scala> distance.reduce((x, y) => if (x(0)>y(0)) x else y)
res4: org.apache.spark.mllib.linalg.Vector = [8000010.0]
```

What!? The maximum distance of a taxi drive is 8 million miles!? This gotta be corrupted data... Let's look at a few filter of the data and see which is reasonable

```Bash
scala> Statistics.colStats(distance.filter(_(0) < 1000)).mean
res18: org.apache.spark.mllib.linalg.Vector = [2.8988766702689905]

scala> Statistics.colStats(distance.filter(_(0) < 100)).mean
res19: org.apache.spark.mllib.linalg.Vector = [2.897565939072903]

scala> Statistics.colStats(distance.filter(_(0) < 50)).mean
res20: org.apache.spark.mllib.linalg.Vector = [2.8955079577667737]

scala> Statistics.colStats(distance.filter(_(0) < 20)).mean
res21: org.apache.spark.mllib.linalg.Vector = [2.788128080257221]
```

We see the mean of the dataset is unaffected if we filter the milage to be &lt; 50 miles, so let's only look at these data for now.

```Bash
scala> val dataFiltered = data.sanitize.filter(v => getDistance(v) < 50)

scala> dataFiltered.summary
pickHour dropHour pasCount distance pickLoct         dropLoct           tip
13.547,  13.559,  1.674,   2.902,   -73.974, 40.75,  -73.974,  40.751,  0.497
40.844,  41.741,  1.766,   12.899,  0.006, 0.001,     0.006,  0.002,    0.001
```

Now Everything look good, let's further see how we should partition filtered data

```Bash
scala> distance.count
res23: Long = 10720407                                                          

scala> distance.filter(_(0) < 20).count
res24: Long = 10662998                                                          

scala> distance.filter(_(0) < 10).count
res25: Long = 10136261                                                                                                                     

scala> distance.filter(_(0) < 1.5).count
res28: Long = 4703481

scala> distance.filter(_(0) < 1).count
res26: Long = 2588721
```

We see the data is extremely skewed, with most trip distances being less then the average distance. So for now let's use some intuitive cutoff points and see the difference

```Bash
scala> val shortTrip =  dataFiltered.filter(v => getDistance(v) < 1)
scala> val longTrip =  dataFiltered.filter(v => getDistance(v) > 10)

scala> shortTrip.summary                                                  
pickHour dropHour pasCount distance pickLoct         dropLoct           tip
13.53,   13.567,  1.655,   0.666,   -73.979, 40.754, -73.979,  40.754,  0.496
34.794,  35.026,  1.758,   0.046,   0.015, 0.001,    0.015,  0.002,     0.002

scala> longTrip.summary                                                      
pickHour dropHour pasCount distance pickLoct         dropLoct           tip
13.421,  13.492,  1.722,  15.198,  -73.887,  40.715, -73.93,  40.73,   0.482
40.415,  42.941,  1.806,  17.243,   0.009,  0.005,   0.008,  0.004,    0.008
```

We see long trips appeared to have larger passenger count and departs earlier, both of which hypothesis is intuitive. To learn more about pick up hour

```Bash
scala> shortTrip.pickHour.groupBy(v => v(0)).mapValues(_.size).collect.sortWith((x, y) => x._2 > y._2)
res42: Array[(Double, Int)] = Array((18.0,174637), (19.0,167914), (17.0,149969), (15.0,148972), (14.0,148370), (12.0,148081), (13.0,143716), (9.0,137585), (11.0,135878), (20.0,135541), (8.0,134112), (16.0,132627), (10.0,130264), (21.0,119631), (22.0,107266), (7.0,94875), (23.0,85598), (0.0,71223), (1.0,54883), (6.0,53013), (2.0,41327), (3.0,30096), (5.0,21663), (4.0,21480))

scala> longTrip.pickHour.groupBy(v => v(0)).mapValues(_.size).collect.sortWith((x, y) => x._2 > y._2)
res43: Array[(Double, Int)] = Array((14.0,36874), (15.0,36247), (16.0,32898), (17.0,31912), (22.0,30992), (21.0,30692), (20.0,30292), (13.0,30137), (18.0,28705), (23.0,28314), (19.0,27600), (12.0,26001), (10.0,23614), (11.0,22982), (7.0,21469), (9.0,21325), (6.0,21308), (8.0,20183), (0.0,20019), (5.0,15693), (1.0,12565), (4.0,10980), (2.0,8792), (3.0,8211))
```

The peak our for taking off varies greatly. It says if one is taking a long distance trip, they most likely departs in afternoon so that they arrive at destination earlier.

