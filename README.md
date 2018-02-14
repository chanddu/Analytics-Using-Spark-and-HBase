# Analytics Using Spark and HBase

In this assignment I used Spark to connect to data stored in HBase tables and ran analytical queries. Since HBase is not avilable in UTD cluster, I used Cloudera's docker container. This assignment has following steps.

### Step I:
1. Download the bike sharing dataset from: http://www.utdallas.edu/~axn112530/cs6350/data/bikeShare/201508_trip_data.csv <br>
Hint: On the UNIX shell, you can run the following <br>
curl –o 201508_trip_data.csv http://www.utdallas.edu/~axn112530/cs6350/data/bikeShare/201508_trip_data.csv <br>
2. Analyze the data and look at the fields. Check if it has a header. Create table and at least one column family in HBase so that this data can be imported. You can do this using the command line or using the Hue GUI.<br>
3. Import the data into the table that you created in step 2. You can do this using any of the Hadoop technologies, such as Pig or Spark. An example of this was shown in the class.<br>
4. Make sure that the data has been imported correctly by looking at it on the Hue GUI.<br>

### Step II:
1. In this step, I used Spark to connect to the HBase table that I created in step I. Below are some hints:
* Download the Spark HBase connector jar file from: https://github.com/nerdammer/spark-hbase-connector The above page also contains helpful hints and code snippets. You can directly download the jar file as: 
`curl -o spark-hbase-connector.jar http://central.maven.org/maven2/it/nerdammer/bigdata/spark-hbase-connector_2.10/0.9.2/spark-hbase-connector_2.10-0.9.2.jar`
* When starting Spark shell use the following command: `spark-shell --jars spark-hbase-connector.jar`
* On the first line of the Spark shell, import the library as: `import it.nerdammer.spark.hbase._`


### Commands used to load and store and connect spark to Hbase
```
T = LOAD '/user/cloudera/201508_trip_data.csv' USING PigStorage(',') AS(
		trip_id:chararray,
		duration:chararray,
		start_date:chararray,
		start_station:chararray,
		start_terminal:chararray, 
		end_date: chararray,
		end_station: chararray,
		end_terminal: chararray,
		bikeno:chararray,
		subscriber_type:chararray,
		zipcode:chararray);

STORE T INTO 'hbase://trip_data' USING org.apache.pig.backend.hadoop.hbase.HBaseStorage(
'data:duration,
data:start_date,
data:start_station,
data:start_terminal, 
data:end_date,
data:end_station,
data:end_terminal,
data:bikeno,
data:subscriber_type,
data:zipcode'
);

curl -o spark-hbase-connector.jar http://central.maven.org/maven2/it/nerdammer/bigdata/spark-hbase-connector_2.10/0.9.2/spark-hbase-connector_2.10-0.9.2.jar 

spark-shell --jars spark-hbase-connector.jar
```

#### Following Queries were answered as follows:

* List the top 10 most popular start stations i.e. those start stations that have the highest count in the dataset
```
val mostPopularStartStations = sc.hbaseTable[(String)]("trip_data").select("start_station").inColumnFamily("data").map(x=>(x,1)).reduceByKey(_+_).sortBy(-_._2).take(10)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q1.png)

* List the top 10 most popular end stations i.e. those end stations that have the highest count in the dataset
```
val mostPopularEndStations = sc.hbaseTable[(String)]("trip_data").select("end_station").inColumnFamily("data").map(x=>(x,1)).reduceByKey(_+_).sortBy(-_._2).take(10)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q2.png)

* List the top 10 start stations that have the highest average trip duration
```
val dRDD = sc.hbaseTable[(String)]("trip_data").select("duration").inColumnFamily("data").withStartRow("432947").withStopRow("913461")
val sSRDD = sc.hbaseTable[(String)]("trip_data").select("start_station").inColumnFamily("data").withStartRow("432947").withStopRow("913461")
val sql_context= new org.apache.spark.sql.SQLContext(sc)
val jRDD = sSRDD.zip(dRDD).map{case ((a),(b)) => (a,b.toDouble)}
val AvderageDF = sql_context.createDataFrame(jRDD)
val result = AvderageDF.groupBy("_1").agg(avg(AvderageDF("_2")))
result.sort($"avg(_2)".desc).take(10).foreach(println)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q3.png)

* Which zip code has the highest number of stations (you can take either start or end stations)
```
val zRDD = sc.hbaseTable[(String)]("trip_data").select("zipcode").inColumnFamily("data")
val sSRDD = sc.hbaseTable[(String)]("trip_data").select("start_station").inColumnFamily("data")
val jRDD = zRDD.zip(sSRDD).map{case ((a),(b)) => (a,b)}.distinct().filter(x => x._1!="nil")
val resultRDD = jRDD.map{case(a,b)=>(a,1)}.reduceByKey((a,b)=>(a+b)).sortBy(-_._2).take(1).foreach(println)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q4.png)

* What is the average duration of the trips that start from any station that contains 'San Francisco' in their name
```
val dRDD = sc.hbaseTable[(String)]("trip_data").select("duration").inColumnFamily("data").withStartRow("432947").withStopRow("913461")
val sSRDD = sc.hbaseTable[(String)]("trip_data").select("start_station").inColumnFamily("data").withStartRow("432947").withStopRow("913461")
val sql_context= new org.apache.spark.sql.SQLContext(sc)
val jRDD = sSRDD.zip(dRDD).map{case ((a),(b)) => (a,b.toDouble)}
val AvderageDF = sql_context.createDataFrame(jRDD)
AvderageDF.groupBy("_1").agg(avg(AvderageDF("_2"))).filter($"_1".contains("San Francisco")).foreach(println)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q5.png)

* Give the breakdown of subscriber type of users and the count of their occurrence
```
val sTRDD = sc.hbaseTable[(String)]("trip_data").select("subscriber_type").inColumnFamily("data")
val sRDD = sTRDD.filter(line=>line(0).contains("Subscriber")).map(line=>line(0),1).reduceByKey((a,b)=>a+b)
val customerRDD = sTRDD.filter(line=>line(0).contains("Customer")).map(line=>line(0),1).reduceByKey((a,b)=>a+b)

sRDD.collect().foreach(println)
customerRDD.collect().foreach(println)
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q6.png)

* Give summary statistics for the duration column e.g. count, min, max, mean, stddev
```
val sql= new org.apache.spark.sql.SQLContext(sc)
val duration = sc.hbaseTable[(String)]("trip_data").select("duration").inColumnFamily("data").filter(row=> row forall Character.isDigit).map(x => (x.toInt,1))
val durationRDD = sql.createDataFrame(duration)
durationRDD.describe("_1").show()
```
![](https://github.com/chanddu/Analytics-Using-Spark-and-HBase/blob/master/Output%20Screen%20Shots/q7.png)
