# CS 598: Cloud Computing Capstone Task 1 by Harrison Kiang - ([Video Link: https://mediaspace.illinois.edu/media/t/1_dly65rlw](https://mediaspace.illinois.edu/media/t/1_dly65rlw))


Note: If you're trying to run this, be sure to edit the `allArgs` variable in [build.gradle](build.gradle)!!!

This is part of the Capstone Project of
the Summer 2018 offering of UIUC's
Cloud Computing Capstone course
as a part of UIUC's MCS-DS program. Details can be found in the [Capstone Project Overview](https://www.coursera.org/learn/cs-598-ccc/supplement/ZSome/capstone-project-overview).

Documented is a data cleaning process in addition to a series of Hadoop tasks and Cassandra queries to answer questions about the [Transporation Databases](https://aws.amazon.com/datasets/transportation-databases/).
Questions answered included top 10 airports/airlines for certain metrics, as well as finding flights under certain constraints.

## Introduction

There are multiple parts to this task.
The first part includes notes on decisions made to set up the infrastructure.
Next includes notes on how the data cleaning process is structured.
The main portion of the paper includes documentation surrounding the Hadoop jobs set up to answer the questions listed in the project overview.
Finally there are notes on how Cassandra was integrated to answer queries for Group 3 question 2.

## Setup

### Hadoop

The following was tested using a custom Hadoop 3.1.0 cluster using Ubuntu 16.04 LTS AMIs on an AWS c4.2xlarge Namenode and 2 c4.xlarge Datanodes.
This setup provides the ability to run Hadoop binaries natively and the ability to run Cassandra as a service from an Ubuntu package.
As such, Ubuntu makes for a great native desktop environment upon which to perform local development.

Initial development was performed locally. After the code was proven out locally, it was ported to the Hadoop cluster and tweaked to be able to run on said cluster.

This [guide](https://github.com/tzyychin/Hadoop_3.0.3_on_Amazon_EC2/blob/master/NameNode.md) on setting up a Hadoop cluster on AWS works for version 3.1.0. Be sure to perform operations for additional datanodes!

### Cassandra
Cassandra was installed as a service in Ubuntu.

```
sudo service cassandra start
```

This allowed for the creation of a Java client to easily interface with Cassandra, as an instance can be reached on the `localhost`.
Given more AWS credits, a distributed Cassandra setup could have been set up.

`nodetoool status` tells whether or not Casasndra is running.
When Cassandra is running, the output result from `nodetool status` is shown below:

```
$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns (effective)  Host ID                               Rack
UN  127.0.0.1  281.31 KiB  256          100.0%            2b8acd64-0879-4c03-9298-db81ff11144a  rack1
```

#### Cassandra Keystore

A lot of this was inspired by this [blog post](http://www.baeldung.com/cassandra-with-java) on how to use Cassandra with Java.
I am using Cassandra 3.5.0.

A `KeyspaceRepository` class was created as per the above blog post.
It essentially is a wrapper around the following statement:

```
CREATE KEYSPACE IF NOT EXISTS airline_ontime WITH replication = {'class':'SimpleStrategy','replication_factor':1};
```

There's also some functionality to drop the keyspace.
Tests were written in `KeyspaceRepositoryTest` to test expected behavior.

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/cassandra/KeyspaceRepository.java`
- `src/test/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/cassandra/KeyspaceRepositoryTest.java`

### Gradle

I used `gradle` to manage dependencies, allowing me to have tight control over which features from Hadoop I wished to used, e.g., certain implementations in `org.apache.hadoop.mapreduce.lib.input.MultipleInputs.MultipleInputs` used in Question 3.2.
This flexibility extends the freedom to use other libraries as well to more easily integrate systems such as:
- Apache commons
- guava
- Cassandra
- anything else I found convenient to use

Gradle also governs the execution of tasks, as its `JavaExec` interface provides an easy to use method of executing specific Java files.

## Dataset

The dataset [Transporation Databases](https://aws.amazon.com/datasets/transportation-databases/) was mounted to an EBS volume and attached to the Namenode of the Hadoop cluster.
The `airline_ontime` directory was copied to the root volume in a pre-defined location accessible by the code, gradle tasks.
The data volume was unmounted.

Files in `airline_ontime` were recusively unzipped in parallel, and the `.zip` files were removed using the following script:
    ```bash
    cd /path/to/task1/airline_ontime
    find . -name "*.zip" | xargs -P 5 -I fileName sh -c 'unzip -o -d "$(dirname "fileName")/$(basename -s .zip "fileName")" "fileName"'
    ```
More information on this script can be found [here]( https://stackoverflow.com/questions/107995/how-do-you-recursively-unzip-archives-in-a-directory-and-its-subdirectories-from
).

In the end, files existed in the following form:
```
/path/to/airline_ontime/2008/On_Time_On_Time_Performance_2008_1/2008/On_Time_On_Time_Performance_2008_1.csv
```

There were 240 `.csv` files of interest in the end, each corresponding to year/month from 1988 to 2008.

## Data Cleaning

For each question, there exists its very own cleaning step and Hadoop job.
The reason why there exists a cleaning job per Hadoop job is that the cleaned data for one question may not be appropriate for another, e.g., due to missing or malformed fields specific to one question or another.

While there are many `.csv` files, one per year/month, there should only exist a single `.txt` file for each Hadoop task.
A class called `PrepareData` was created that iterates through all of the extracted `.csv` files extracted from the `.zip` archives in the `airline_ontime` dataset, and extracts the desired columns to keep to store in a single `.txt` file for the corresponding Hadoop job to consume.

Below is an example output prepared by `PrepareData` (pulled from `/data/arrivalsAndDeparturesPerAirport_cleaned_data.txt`):

```java
SEA SFO
MEM MCO
GFK MSP
CVG BUF
MEM MCO
...
```

Optimization: `PrepareData` includes as an optional parameter a `Predicate` which filters which `.csv` files to iterate through.
This dramatically reduces the clean/search space for data relevant to question 3.2 which deals with flight requests for the year 2008.

Another optimization of note: the lines in each `.csv` file are parsed in parallel using Java's parallel streams, reducing data cleaning time for all Hadoop jobs.

## Hadoop jobs

Each Hadoop job cleans the data as describes above, deposits it in HDFS.
It then performs its computation, and outputs its final result in a `part-r-00000` file in the corresponding`.txt` directory.

The reason why the cleaned data is stored locally is that Hadoop jobs can only accept 1-2 files as input at a time.
With this knowledge, parsing each `.csv` source file individually in its own Hadoop job becomes less than feasible.

### Group 1

#### 1. Rank the top 10 most popular airports by numbers of flights to/from the airport.

This was executed in 2 Hadoop job as one generates an output file in alphabetical order by airport code, while the other keeps the top 10 results in descending order by popularity.

The first job cleans the data by only keeping the `Origin` and `Dest` columns from the dataset.
In the `Mapper`, each cleaned row produces two `(airport, 1)` records used for tracking popularity by both departures and arrivals.
The `Reducer` simply aggregates the count per airport.

The second job's `Mapper` is a pass-through to the single `Reducer` that maintains a globaly sorted `Map` of tuples sorted by value.

The top 10 most popular airports are shown below (pulled from `/data/top_10_airports_by_arrivals_and_departures.txt/part-r-00000` in HDFS):

```
ORD	12449354    DEN	6273787
ATL	11540422    DTW	5636622
DFW	10799303    IAH	5480734
LAX	7723596     MSP	5199213
PHX	6585534     SFO	5171023
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group1/ArrivalsAndDeparturesPerAirport.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group1/Top10AirportsByArrivalsAndDepartures.java`

#### 2. Rank the top 10 airlines by on-time arrival performance.

This was executed in 2 separate Hadoop tasks, similar to #1 above.

The first job keeps the `UniqueCarrier` and `ArrDelay` columns from the dataset.
The `Mapper` produces the `(carrier, delay)` tuples.
The `Reducer` averages all of the `delay`s per `carrier`.

The second job's `Mapper` is a pass-through to the single `Reducer` that maintains a globaly sorted `Map` of tuples sorted by value.

The top 10 airlines in ascending order of average arrival delay are shown below (pulled from `/data/top_10_airlines_by_delay.txt/part-r-00000` in HDFS):

```
HA	-1.0118043      F9	5.4658813
AQ	1.1569234       NW	5.5577836
PS	1.4506385       WN	5.5607743
ML(1)	4.747609    OO	5.7363124
PA(1)	5.322431    9E	5.8671846
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group1/AvgDelayPerAirline.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group1/Top10AirlinesByDelay.java`

### Group 2

Since each question answered in Group 2 involves averaging, a `Reducer` called `AvgReducer` was made to facilitate producing an average `Float` per `Text` key.

Likewise, since each question answered in Group 2 involves keeping the top 10 results per key in descending order per float value, a `Reducer` called `Top10ReducerByFloatV1` was created.

#### 1. For each airport X, rank the top-10 carriers in decreasing order of on-time departure performance from X.

Again, 2 Hadoop jobs:
The first job cleans the data by keeping the `Origin`, `UniqueCarrier`, and `DepDelay` columns from the dataset.
The `Mapper` then creates a tuple containing a composite key and value of `(originAndCarrier, depDelay)` to pass to the `AvgReducer`.
Note that the composite key has a space so that the output will appear as intended.

The top 10 carriers in decreasing order of departure delay per airport are shown below for CMI, BWI, MIA, LAX, IAH, and SFO (pulled from `/data/top_10_carriers_by_dep_delay_per_airport.txt/part-r-00000` in HDFS):

```
CMI OH	0.61162645      BWI F9	0.75624377
CMI US	2.0330474       BWI PA(1)	4.7619047
CMI TW	4.1206155       BWI CO	5.179341
CMI PI	4.4556284       BWI YV	5.4965034
CMI DH	6.0278883       BWI NW	5.705573
CMI EV	6.665138        BWI AA	6.002852
CMI MQ	8.016005        BWI 9E	7.2398057
                        BWI US	7.4943957
                        BWI DL	7.676822
                        BWI UA	7.737921

MIA 9E	-3.0            LAX MQ	2.4072218
MIA EV	1.2026432       LAX OO	4.221959
MIA TZ	1.7822436       LAX FL	4.725127
MIA XE	1.8731909       LAX TZ	4.763941
MIA PA(1)	4.200004    LAX PS	4.8603373
MIA NW	4.5016656       LAX NW	5.1195507
MIA US	6.090666        LAX F9	5.7291555
MIA UA	6.869732        LAX HA	5.813646
MIA ML(1)	7.50455     LAX YV	6.024156
MIA FL	8.565107        LAX US	6.7463956

IAH NW	3.5637107       SFO TZ	3.9524157
IAH PA(1)	3.9847274   SFO MQ	4.853924
IAH PI	3.9886668       SFO F9	5.1624446
IAH US	5.0602674       SFO PA(1)	5.2876115
IAH F9	5.5452437       SFO NW	5.757806
IAH AA	5.703959        SFO PS	6.303519
IAH TW	6.0487776       SFO DL	6.56273
IAH WN	6.2311335       SFO CO	7.0830493
IAH OO	6.5879583       SFO US	7.52751
IAH MQ	6.7129736       SFO TW	7.794883
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/AvgDepDelayPerAirportCarrier.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/Top10CarriersByDepDelayPerAirport.java`

##### AirportCarrierDepDelayRepository for Cassandra

Interaction with the above Cassandra setup is facilitated by `CassandraConnector`.
`Airport` and `AirportCarrierDepDelay` Data Transfer Objects (DTOs) were created to facilitate the processing of records into and out of Cassandra.
They have the following class definitions:

```
Airport(String airport)
AirportCarrierDepDelay(UUID id, String airport, String uniqueCarrier, double depDelay)
```
See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/cassandra/CassandraConnector.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question1/AirportCarrierDepDelay.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question1/Airport.java`

The `AirportCarrierDepDelayRepository` class was created to control the creation, update, and queries of the `airportCarrierDepDelay` table, which contains the `AirportCarrierDepDelay` records.
The schema for the table controlled by `AirportCarrierDepDelayRepository` is below:

```
CREATE TABLE IF NOT EXISTS airline_ontime.airportcarrierdepdelay()
id uuid PRIMARY KEY,
airport text,
unique_carrier text,
dep_delay float,
PRIMARY KEY((airport, unique_carrier), id));
```

Optimization:
The `airportcarrierdepdelay` table's primary key allows for Cassandra to reference an index per query, as each query will look up records in these tables by `(airport)` tuples, i.e., the `Airport`s. 
Queries are implemented using `WHERE` clauses `selectByAirport()` for the optimized table.

Insertions are implemented in `insertAirportCarrierDepDelay()` for the regular table, `insertAirportCarrierDepDelayByAirport()` for the optimized table, and `insertAirportCarrierDepDelayBatch()` for both tables.

Tests are implemented in `AirportCarrierDepDelayRepositoryTest` to check expected behavior including creation of table, insertion of records in both tables, selections, and other queries.
Note that a local running instance of Cassandra must be running in order for the tests to pass.

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question1/AirportCarrierDepDelayRepository.java`
- `src/test/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question1/AirportCarrierDepDelayRepositoryTest.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question1/Airport.java`

#### 2. For each airport X, rank the top-10 airports in decreasing order of on-time departure performance from X.

Much like #1, but replacing `UniqueCarrier` with `Dest`.
The `Mapper`'s composite output key is `originAndDest`.

The top 10 airports in decreasing order of on-time departure performance are shown below for CMI, BWI, MIA, LAX, IAH, and SFO (pulled from `/data/top_10_dest_by_dep_delay_per_airport.txt/part-r-00000` in HDFS):

```
CMI ABI -7.0      BWI SAV -7.0
CMI PIT 1.1024306 BWI MLB 1.1553673
CMI CVG 1.8947617 BWI DAB 1.4695946
CMI DAY 3.1162353 BWI SRQ 1.5884839
CMI STL 3.9816732 BWI IAD 1.7909408
CMI PIA 4.591892  BWI UCA 3.6541698
CMI DFW 5.944143  BWI CHO 3.7449276
CMI ATL 6.665138  BWI GSP 4.1976867
CMI ORD 8.194098  BWI SJU 4.4446583
                  BWI OAJ 4.4711113

MIA SHV 0.0         LAX SDF -16.0
MIA BUF 1.0         LAX IDA -7.0
MIA SAN 1.7103825   LAX DRO -6.0
MIA SLC 2.5371902   LAX RSW -3.0
MIA HOU 2.912199    LAX LAX -2.0
MIA ISP 3.647399    LAX BZN -0.72727275
MIA MEM 3.7451067   LAX MAF 0.0
MIA PSE 3.9758453   LAX PIH 0.0
MIA TLH 4.2614846   LAX IYK 1.2698247
MIA MCI 4.612245    LAX MFE 1.3764706

IAH MSN -2.0        SFO SDF -10.0
IAH AGS -0.6187905  SFO MSO -4.0
IAH MLI -0.5        SFO PIH -3.0
IAH EFD 1.8877082   SFO LGA -1.7575758
IAH HOU 2.172037    SFO PIE -1.3410405
IAH JAC 2.5705884   SFO OAK -0.8132005
IAH MTJ 2.950157    SFO FAR 0.0
IAH RNO 3.2215843   SFO BNA 2.4259665
IAH BPT 3.5995326   SFO MEM 3.3024824
IAH VCT 3.6119087   SFO SCK 4.0

```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question2/AvgDepDelayPerAirportDest.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question2/Top10DestByDepDelayPerAirport.java`

#### 3. For each source-destination pair X-Y, rank the top-10 carriers in decreasing order of on-time arrival performance at Y from X.

Similar to #1 and #2, except the `Mapper`'s composite key contains `originAndDestAndCarrier`.
The top 10 carriers in decreasing order of on-time arrival performance is shown for the following routes (pulled from `/data/top_10_carriers_by_arr_delay_per_airport_pair.txt/part-r-00000` in HDFS):
- CMI → ORD
- IND → CMH
- DFW → IAH
- LAX → SFO
- JFK → LAX
- ATL → PHX

```
CMI ORD MQ	10.143662   DFW IAH PA(1)	-1.5964912
                        DFW IAH EV	5.0925136
IND CMH CO	-2.5458546  DFW IAH UA	5.4142013
IND CMH AA	5.5         DFW IAH CO	6.4937315
IND CMH HP	5.697255    DFW IAH OO	7.5640073
IND CMH NW	5.7615385   DFW IAH XE	8.094295
IND CMH US	6.8784695   DFW IAH AA	8.381228
IND CMH DL	10.6875     DFW IAH DL	8.598509
IND CMH EA	10.813084   DFW IAH MQ	9.103211

LAX SFO TZ	-7.6190476  JFK LAX UA	3.3138745
LAX SFO PS	-2.1463416  JFK LAX HP	6.680599
LAX SFO F9	-2.0286858  JFK LAX AA	6.9037247
LAX SFO EV	6.96463     JFK LAX DL	7.93446
LAX SFO AA	7.3867936   JFK LAX PA(1)	11.0194435
LAX SFO MQ	7.8077636   JFK LAX TW	11.702008
LAX SFO US	7.964722
LAX SFO WN	8.792051    ATL PHX FL	4.5526314
LAX SFO CO	9.354783    ATL PHX US	6.288115
LAX SFO NW	9.848785    ATL PHX HP	8.481437
                        ATL PHX EA	8.953571
                        ATL PHX DL	9.808275
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question2/AvgArrDelayPerAirportPairCarrier.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group2/question2/Top10CarriersByArrDelayPerAirportPair.java`

### Group 3

#### 1. Does the popularity distribution of airports follow a Zipf distribution? If not, what distribution does it follow?

There are two phasses to answering this problem:
1. Getting the popularity of each airport in descending order by popularity (defined as number of arrivals and departures)
2. Fitting different power law distributions to the data and seeing which one fits best.

##### Phase 1
The popularity for each airport was determined by a Hadoop task that reads from the results of Group 1 #1 in `data/arrivals_and_departures_per_airport.txt/part-r-00000` in HDFS.

A Hadoop job was used to sort through these results.
The `Mapper` created `(freq, NullWritable)` values.
A `WritableComparator` called `DescendingIntegerComparator` was written to compare each key (the freq) in descending order.
Note that there is effectively no "value" passed from the `Mapper`.
The default `Reducer` was used as a pass through to write to a `.txt` file.

A sample of the sorted results are shown below (pulled from `/data/arrivals_and_departures_per_airport_sorted_by_frequency.txt/part-r-00000` in HDFS):

```
12449354 6273787
11540422 5636622
10799303 5480734
7723596 5199213
6585534 5171023
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/ArrivalsAndDeparturesPerAirportSortedByFrequency.java`

##### Phase 2

Multiple power law distributions were fit to the data and visualized via Matplotlib.
Data is read from the output of the Hadoop job in Phase 1 and charted on log-log axes.
The x axis represents the rank, while the y axis represents popularity (number of arrivals and departures).

Each of the power law distributions were fit to the data using `scipy.optimize.curve_fit`, and then scaled
such that the first elements in the data and the fit curve are of the same magnitude and are well represented in the log-log axes.
The zipf, poisson, lognormal, and pareto distributions are charted below:

![](https://raw.githubusercontent.com/hkiang01/Cloud-Computing-Capstone-Task-1/master/docs/Figure_1.png?token=AFMhnU2wPlpz1qa3qL86A5VYJmmVb2yDks5bRBRPwA%3D%3D "Group 3 Question 1 plot")

The blue dots represent the popularity distribution.

The reason why the graph was charted in log-log is because of the ease to compare the results to the Zipf distribution.
>  Zipf's law is most easily observed
      by plotting the data on a log-log graph,
      with the axes being log (rank order) and log (frequency). [...]
      The data conform to Zipf's law to the extent that the plot is linear.
    - [Zipf's Law - Wikipedia](https://en.wikipedia.org/wiki/Zipf%27s_law)

If the popularity distribution fits the Zipf distribution, then it should also follow a linear line from the upper left
to the lower right of the graph with log-log axes.
This is not the case.
Of all the charted distributions, the popularity distirbution most closely follows the Poisson distribution.

See:
- `edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/zipf.py`

#### 2. Tom wants to travel from airport X to airport Z. However, Tom also wants to stop at airport Y for some sightseeing on the way. More concretely, Tom has the following requirements (for specific queries, see the Task 1 Queries and Task 2 Queries)

a) The second leg of the journey (flight Y-Z) must depart two days after the first leg (flight X-Y). For example, if X-Y departs on January 5, 2008, Y-Z must depart on January 7, 2008.

b) Tom wants his flights scheduled to depart airport X before 12:00 PM local time and to depart airport Y after 12:00 PM local time.

c) Tom wants to arrive at each destination with as little delay as possible. You can assume you know the actual delay of each flight.

This was accomplished in 4 phases:
1. Collect all the available airports
2. Generate the ability to make requests of type `(originAirport, stopAirport, destAirport, date)`
3. Format the leg candidates so that they can be joined against the requests
4. "Join" each request with the best leg candidates while meeting the above constraints.

##### Phase 1
The listing of airports was accomplished through a Hadoop job.

The data was cleaned by only keeping the `Origin` and `Dest` columns from the dataset.
In the `Mapper`, each cleaned row produces two `(airport, null)` records.
For each key or airport, the `Reducer` simply outputs a single record, that same airport.

See:
`src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/Airports.java`

##### Phase 2

The challenge is to find legs for Tom's trip for any hypothetical request between three not necessarily distinct airports (origin and destination could be the same) for all dates in 2008.
In order for these requests to be processed by Hadoop, they must be generated and stored in a file.

We start with generating `(origin, stop, dest)` tuples.
This was done through a Hadoop job that uses the [`combinatorics3`](https://github.com/dpaukov/combinatoricslib3), a combinatorics library for Java 8.

The `Mapper` simply passes the airports processed in Phase 1.
The `Reducer` uses the [`combinatorics3`](https://github.com/dpaukov/combinatoricslib3) library to generate the valid permutations (`stop` cannot be the same as `origin` or `dest`).

Optimization:
The ability for the [`combinatorics3`](https://github.com/dpaukov/combinatoricslib3) package to generate variations or permutations with repetition into a Java 8 stream allowed for parallel writes to a text file, speeding up the generation of `(origin, stop, dest)` tuples.

Sample airport triplets are shown below (pulled from `/data/origin_stop_dest.txt/part-r-00000` in HDFS):

```
CYS SIT YUM
CWA SIT YUM
CVG SIT YUM
CSG SIT YUM
CRW SIT YUM
CRP SIT YUM
...
```

- see:
`src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/OriginStopDest.java`

##### Phase 3

For each permutation triplet, 366 requests were generated and stored in a `.txt` file in `(airportOrigin, airportStop, airportDestination, yyyy-MM-dd)` tuples, one for each day in 2008.

Sample requests are shown below (pulled from `/data/requests.txt/part-r-00000` in HDFS):

```
JAX ABY ABE 2008-10-23
CMI ABI ABE 2008-03-03
JAX ABY ABE 2008-10-24
BFL ADQ ABE 2008-06-26
CMI ABI ABE 2008-03-04
JAX ABY ABE 2008-10-25
```

Note that a limit for the number of generated requests was used for local development so as not to completely fill the local disk.

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/Requests.java`

##### Phase 4

The leg candidates must be formatted properly to be "joined" against the requests and produce output like that shown in the examples, which include airline, scheduled departure time in the `HH:mm dd/MM/yyyy` format, etc.

This is done through a Hadoop job.
The cleaned data preserves the following columns from the dataset: `Origin`, `Dest`, `FlightDate`, `UniqueCarrier`, `FlightNum`, `DepTime`, `DepDelay`, `ArrDelay`.

The `Mapper` formats the data appropriately, taking care to only write records for `FlightDate`s in the year 2008.
It also takes care to format the dates accordingly, as some records have flight dates with hours greater than 23, in which case a day is added to attempt to rectify the malformed date.
Finally, it calculates the scheduled departure time by subtracting the departure delay from the local departure time.


The `Reducer` simply writes the received `(origin, dest, flightDate, flightNum, scheduledDeparture, arrDelay)` tuples to the output file.

Sample leg candidates are shown below (pulled from `/data/leg_candidates.txt/part-r-00000` in HDFS):

```
ORD JFK 2008-08-16 AA 2464 16:55 16/08/2008 17.00
ORD JFK 2008-08-16 B6 900 06:00 16/08/2008 16.00
ORD JFK 2008-08-16 B6 904 09:35 16/08/2008 -13.00
ORD JFK 2008-08-16 B6 908 11:55 16/08/2008 -14.00
ORD JFK 2008-08-16 B6 916 16:05 16/08/2008 98.00
ORD JFK 2008-08-16 MQ 4138 09:50 16/08/2008 -11.00
ORD JFK 2008-08-16 OH 5234 17:30 16/08/2008 3.00
ORD JFK 2008-08-16 OH 5368 11:40 16/08/2008 15.00
ORD JFK 2008-08-16 OH 5688 09:35 16/08/2008 1.00
```

See:
`src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/LegCandidates.java`

##### Phase 5

Finding the best leg candidates per request is done through a "join" operation by using Hadoop jobs, one per each set of constraints.
The "joins" themselves are actually implemented through the use of the `MultipleInputs` class in the Hadoop mapreduce library. The programming model is as follows:

There are 2 `Mapper`s: one for the requests (`RequestsMapper`), and another for the leg candidates (`LegCandidatesMapper`).
- `RequestsMapper` maps each reqeust to a group: `(origin, stop, date)` for leg 1 and `(stop, dest, date)` for leg 2, with 2 days subtracted from the flight date to satisfy constraint a.
The entirety of the request, `(origin, stop, dest, date)`, is forwarded as the value for downstream processing (data queries in Cassandra), preceded by the string "request" that is used by the `Reducer`.
- `LegCandidatesMapper` maps each leg candidate to a group: `(origin, dest, date)`.
The entirety of the leg candidate record, `(origin, dest, uniqueCarrier, flightNum, scheduledDep, arrDelay)` is forwarded as the value for downstream processing, preceded by the string "leg1" or "leg2" that is used by the `Reducer`.
Note that only candidates before 12:00 for leg 1 and after 12:00 for leg 2 are passed onto the `Reducer` to satisfy constraint b.

The `ReduceJoinReducer` collects all of the values per group: the request, preceded by the "request" string, and the leg candidates, preceded by the strings "leg1" or "leg2".
If there exists a request for the group and there is at least one leg candidate, then request and the leg candidate with the least arrival delay is written along with the request to a `.txt` file to satisfy constraint c.

Sample queries for leg 1 are shown below (pulled from `/data/toms_legs1.txt/part-r-00000` in HDFS):

```
request	CMI ORD LAX 2008-03-04 leg1	CMI ORD MQ 4278 07:10 04/03/2008 -14.00
request	DFW ORD DFW 2008-06-10 leg1	DFW ORD UA 1104 07:00 10/06/2008 -21.00
request	JAX DFW CRP 2008-09-09 leg1	JAX DFW AA 845 07:25 09/09/2008 1.00
request	LAX ORD JFK 2008-01-01 leg1	LAX ORD UA 944 07:05 01/01/2008 1.00
request	LAX SFO PHX 2008-07-12 leg1	LAX SFO WN 3534 06:50 12/07/2008 -13.00
request	SLC BFL LAX 2008-04-01 leg1	SLC BFL OO 3755 11:00 01/04/2008 12.00
```

Sample queries for leg 2 are shown below (pulled from `/data/toms_legs2.txt/part-r-00000`):

```
request	SLC BFL LAX 2008-04-01 leg2	BFL LAX OO 5429 14:55 03/04/2008 6.00
request	JAX DFW CRP 2008-09-09 leg2	DFW CRP MQ 3627 16:45 11/09/2008 -7.00
request	DFW ORD DFW 2008-06-10 leg2	ORD DFW AA 2341 16:45 12/06/2008 -10.00
request	LAX ORD JFK 2008-01-01 leg2	ORD JFK B6 918 19:00 03/01/2008 -7.00
request	CMI ORD LAX 2008-03-04 leg2	ORD LAX AA 607 19:50 06/03/2008 -24.00
request	LAX SFO PHX 2008-07-12 leg2	SFO PHX US 412 19:25 14/07/2008 -19.00
```

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/TomsLegs1.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/TomsLegs2.java`


### RequestLegRepository

Interaction with the above Cassandra setup is facilitated by `CassandraConnector`.
`Request` and `RequestLeg` Data Transfer Objects (DTOs) were created to facilitate the processing of records into and out of Cassandra.
They have the following class definitions:

```
Request(UUID id, String request_origin, String request_stop, String request_dest, LocalDate request_date);

RequestLeg(UUID id, String request_origin, String request_stop, String request_dest, LocalDate request_date, int leg_num, String leg_origin, String leg_dest, String leg_unique_carrier, int leg_flight_num, LocalDateTime leg_scheduled_departure, int leg_arr_delay);
```
See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/cassandra/Request.java`
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/cassandra/RequestLeg.java`

The `RequestLegRepository` class was created to control the creation, update, and queires of the `requestLegs` table, which contains the `RequestLeg` records.
The schema for the table controlled by `RequestLegRepository` is below:

```
CREATE TABLE IF NOT EXISTS airline_ontime.requestlegsbyrequest()
id uuid,
request_origin text,
request_stop text,
request_dest text,
request_date timestamp,
leg_num int,
leg_origin text,
leg_dest text,
leg_unique_carrier text,
leg_flight_num int,
leg_scheduled_departure timestamp,
leg_arr_delay int,
PRIMARY KEY((request_origin, request_stop, request_dest, request_date), id));
```

Optimization:
The `requestlegsbyrequest` table's primary key allows for Cassandra to reference an index per query, as each query will look up records in these tables by `(origin, stop, dest, date)` tuples, i.e., the `Request`s. 
Queries are implemented using `WHERE` clauses `selectByRequest(Request)` for the optimized table.

Insertions are implemented in `insertRequestLeg(RequestLeg)` for the regular table, `insertRequestLegByRequest(RequestLeg)` for the optimized table, and `insertRequestLegBatch(RequestLeg)` for both tables.

Tests are implemented in `RequestLegTest` to check expected behavior including creation of table, insertion of records in both tables, selections, and other queries.
Note that a local running instance of Cassandra must be running in order for the tests to pass.

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/cassandra/RequestLegRepository.java`
- `src/test/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/cassandra/RequestLegTest.java`

### CassandraClient

FYI: this isn't part of the workflow, it's functionally just a diagnostic tool used to output results for test requests.

`RequestsClientToCassandra` simply takes records from `/data/toms_legs1.txt/part-r-00000` and `/data/toms_legs2.txt/part-r-00000` in HDFS,
maps them to `RequestLeg`s, and inserts them into the `airline_ontime.requestlegsbyrequest` table in Cassandra.
Below are the records made available in Cassandra.


```
$ cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.2 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> SELECT * FROM airline_ontime.requestlegsbyrequest;

 request_origin | request_stop | request_dest | request_date                    | id                                   | leg_arr_delay | leg_dest | leg_flight_num | leg_num | leg_origin | leg_scheduled_departure         | leg_unique_carrier
----------------+--------------+--------------+---------------------------------+--------------------------------------+---------------+----------+----------------+---------+------------+---------------------------------+--------------------
            CMI |          ORD |          LAX | 2008-03-04 05:00:00.000000+0000 | db4947db-333f-4c2e-8141-4aaf258acb6e |           -14 |      ORD |           4278 |       1 |        CMI | 2008-03-04 12:10:00.000000+0000 |                 MQ
            CMI |          ORD |          LAX | 2008-03-04 05:00:00.000000+0000 | f5f26be5-7be1-42e0-ba1d-935476e01799 |           -24 |      LAX |            607 |       2 |        ORD | 2008-03-07 00:50:00.000000+0000 |                 AA
            LAX |          SFO |          PHX | 2008-07-12 04:00:00.000000+0000 | a3fb052c-eab5-4606-8d1c-3909eb1c962f |           -19 |      PHX |            412 |       2 |        SFO | 2008-07-14 23:25:00.000000+0000 |                 US
            LAX |          SFO |          PHX | 2008-07-12 04:00:00.000000+0000 | e8606b06-d904-4345-a5f9-de9e7bdab45e |           -13 |      SFO |           3534 |       1 |        LAX | 2008-07-12 10:50:00.000000+0000 |                 WN
            JAX |          DFW |          CRP | 2008-09-09 04:00:00.000000+0000 | 52366f53-df3d-4c29-8116-472cc2e5b9d8 |            -7 |      CRP |           3627 |       2 |        DFW | 2008-09-11 20:45:00.000000+0000 |                 MQ
            JAX |          DFW |          CRP | 2008-09-09 04:00:00.000000+0000 | bb5f72d0-384e-400f-a777-fca2112cbb29 |             1 |      DFW |            845 |       1 |        JAX | 2008-09-09 11:25:00.000000+0000 |                 AA
            DFW |          ORD |          DFW | 2008-06-10 04:00:00.000000+0000 | 75d9ca2e-8f4a-4f58-88fe-e181ecff6ccf |           -21 |      ORD |           1104 |       1 |        DFW | 2008-06-10 11:00:00.000000+0000 |                 UA
            DFW |          ORD |          DFW | 2008-06-10 04:00:00.000000+0000 | fe038fb1-81e4-42a1-b37f-a3632a90f131 |           -10 |      DFW |           2341 |       2 |        ORD | 2008-06-12 20:45:00.000000+0000 |                 AA
            SLC |          BFL |          LAX | 2008-04-01 04:00:00.000000+0000 | 206fc61b-7a8a-4f47-8a8f-f3b57b01f054 |            12 |      BFL |           3755 |       1 |        SLC | 2008-04-01 15:00:00.000000+0000 |                 OO
            SLC |          BFL |          LAX | 2008-04-01 04:00:00.000000+0000 | 81dca602-f7bc-4186-84ee-e67f3a78d388 |             6 |      LAX |           5429 |       2 |        BFL | 2008-04-03 18:55:00.000000+0000 |                 OO
            LAX |          ORD |          JFK | 2008-01-01 05:00:00.000000+0000 | 2e1e0d6a-87eb-4e62-b5ab-f2211801f082 |            -7 |      JFK |            918 |       2 |        ORD | 2008-01-04 00:00:00.000000+0000 |                 B6
            LAX |          ORD |          JFK | 2008-01-01 05:00:00.000000+0000 | 6009f9f0-907f-42f5-bca2-b443c3670daf |             1 |      ORD |            944 |       1 |        LAX | 2008-01-01 12:05:00.000000+0000 |                 UA
```

Sample stdout for `RequestsClientToCassandra` when run against `testRequests`:

See:
- `src/main/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/RequestsClientToCassandra.java`
- `whenCreatingARequestLeg_thenCreatedCorrectly` in `src/test/java/edu/illinois/hkiang2/mcsds/cloud_computing_capstone/task1/group3/cassandra/RequestLegTest.java`
