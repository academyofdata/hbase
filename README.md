# Apache HBase basics
## Preparation of data 
start by downloading the data files in local file system
```
export OUTDIR=.
wget https://raw.githubusercontent.com/academyofdata/data/master/movies-with-year.csv -O $OUTDIR/movies.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings2.csv -O $OUTDIR/ratings_s.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings.csv.gz -O $OUTDIR/ratings.csv.gz
gunzip -f $OUTDIR/ratings.csv.gz
wget https://raw.githubusercontent.com/academyofdata/data/master/users.csv -O $OUTDIR/users.csv
```
## We will use HBase utility ImportTsv to import data from the denormalized ratings file. But before we need to build 
## the data accordingly: Our key will be movieid:userid and we will have 3 column families.  
use bash and some utilities to prepare files in a format that's suitable for importtsv 
```
awk -F, -v u=1 -v m=8 -v OFS="," -v ORS="" '{print $m":"$u",";for(i=1;i<=NF;i++)printf("%s%s",$i,(i!=NF)?OFS:"\n")}' $OUTDIR/ratings.csv | tail -n +2 > $OUTDIR/ratings1.csv

```
executing the script on the line above has the effect of creating a slightly modified version of the denormalized rating file ratings.csv, i.e. each line is augmented with a first field that's the composition of the movieid and userid, which looks like this
```
4637:31,4637,35,F,9,48302,3.0,2000-07-19 18:48:41,31,Dangerous Minds (1995),1995,{Drama}
3841:31,3841,45,M,18,26101,3.0,2000-08-11 13:14:31,31,Dangerous Minds (1995),1995,{Drama}
```
(we've also eliminated the header line by piping through the tail utility)

we can now put the files into HDFS
```
HADOOP_USER_NAME=hdfs hdfs dfs -put $OUTDIR/ratings1.csv /tmp/
```
before loading the data, let's create the HBase table. We will use the HBase shell to create RATINGS1 table with 3 CF's
```
$ hbase shell
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.1.2.2.6.2.0-205, r5210d2ed88d7e241646beab51e9ac147a973bdcc, Sat Aug 26 09:33:50 UTC 2017

hbase(main):001:0> create 'RATINGS1','user','rating','movie'
```
make sure it's created
```
hbase(main):003:0> list 'RATINGS1'
TABLE                                                                                                                       
RATINGS1                                                                                                                    
1 row(s) in 0.0050 seconds
```
and finally use ImportTsv to load the file into HBase, exit the shell and issue the following command
```
HADOOP_USER_NAME=hdfs  hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.separator=,  -Dimporttsv.columns="HBASE_ROW_KEY,user:userid,user:age,user:gender,user:occupation,user:zip,rating:rating,rating:timestamp,movie:movieid,movie:title,movie:year,movie:genres" RATINGS1 hdfs:///tmp/ratings1.csv
```
Please see [here](//hbase.academyofdata.com) for futher exercises on RATINGS1 table. 

## Importing with an HBase shell script
Let's now use the simplified ratings file as an input to transform and load into a separate RATINGS2 table. The RATINGS2 has a single colum family (rating) but each entry has a different column name based on the user that rates the current movie (say 4321), that is the user id is being made part of the column name; so you have rating:rating1234 with value 4.0 to signal that the user 1234 has rated movie 4321 with a rating of 4. The key is the movieid and the entries in 4321 movieid key will show all the ratings and timeline of each rating for this movie. 

```
echo "create 'RATINGS2','rating'" > ratings2.txt
tail -n +2 ratings_s.csv | awk -v tbl=RATINGS2 -F, '{print "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:rating" $1 "'\'','\''" $3 "'\''\n" "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:timestamp" $1 "'\'','\''" $4 "'\''"}' >> ratings2.txt
```
now we have a script that we could pass to hbase shell for execution
```
hbase shell ./ratings2.txt
```
(this will take a while to execute; for a "non-blocking" variant try ```hbase shell  ./ratings2.txt > shell.out 2>&1 &```)

alternatively we could create/import data in a table that has the same kind of information, but with a small model change: instead of naming the columns "rating{userid}" and "timestamp{userid}" inside the column family "rating", we will name them "{userid}rating" and "{userid}timestamp"
```
echo "create 'RATINGS2_1','rating'" > ratings2_1.txt
tail -n +2 ratings_s.csv | awk -v tbl=RATINGS2_1 -F, '{print "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:"$1"rating'\'','\''" $3 "'\''\n" "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:"$1"timestamp'\'','\''" $4 "'\''"}' >> ratings2_1.txt
```
followed by
```
hbase shell ./ratings2_1.txt 
```
to load the data

let's now use the same technique, but this time we would like our column qualifier to be the userid and we limit ourselves to storing just two pieces of information for each rating - the userid (column qualifier) and the rating (value). We will thus use a column that will be called e.g. 310 for the user with the id 310, the value in that column will be the rating of the said user.

```
echo "create 'RATINGS3','rating'" > ratings3.txt
tail -n +2 ratings_s.csv | awk -v tbl=RATINGS3 -F, '{print "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:" $1 "'\'','\''" $3 "'\''"}' >> ratings3.txt
```
then use a shell to load it into HBase
```
hbase shell ./ratings3.txt
```
We have now three tables RATINGS2, RATINGS2_1, RATINGS3 that have MOVIEDID as row key but different column arrangements:
* RATINGS2 has column qualifiers: ratings+userid (value rating) and ratings+TS (value timestamp)
* RATINGS2_1 has column qualifiers: userid+rating (value rating) and timestamp+rating (value timestamp)
* RATINGS3 has column qualifier: userid and value rating. 

Please see [here](//hbase.academyofdata.com) for futher exercises on RATINGS1 table.   

## ROWKEY management. 
## distributing the writes
Let's assume we'll want to lookup these ratings by the time they were given, rather than by movieid (as it was the case so far). This means we should include the timestamp as the first part of the ROWKEY. The simplest way would be to transform the input file using the following line, then using ImportTsv to load it 
```
tail -n +2 ratings_s.csv | awk -F, '{print $4":"$2","$1","$3}' > ratings4.csv
```
Because the timestamps are not uniformly distributed, we have lots of keys starting with 1 and because HBase uses a lexicographic key ordering most of the keys will be handled by the same region server, so we should adopt a salting strategy to distribute the keys more uniformly. For instance let's use the mod 10 of the timestamp as a 'bucket' id (the least significant digit of the timestamp is more uniformly distributed than the most significant one), thus the transformation would look like this
```
tail -n +2 ratings_s.csv | awk -F, '{print $4%10":"$4","$2","$1","$3}' > ratings4.csv
```
this salting method has the advantage that is deterministic (same timestamp will always fall in the same bucket) - this property might come handy when hashing known/non-generated ids (say a systemid). The main disadvantage of the non-deterministic hashing is that any lookup needs to be done in every bucket. That is, assuming that we have a key that has been salted into 10 buckets (just as above) we would need to lookup 0:{x}, 1:{x},....,9:{x}

In this case we could've used something else to salt the key, for instance the line number in the original file - actually the remainder of the division of the line number to 8 (assuming we want 8 'buckets')
```
tail -n +2 ratings_s.csv | awk -F, '{print NR%8":"$4","$2","$1","$3}' > ratings4.csv
```
Another deterministic key management procedure is to use a hash of the key (assuming that the key itself has enough entropy); for instance in the example here, where the key is a timestamp (with enough entropy) we could just sha1 the key (we don't really care that sha1 is not cryptographically secure, we just need a uniform function). So we could do
```
tail -n +2 ratings_s.csv | awk -F, '{print "echo " $4 " | sha1sum|awk '\''{printf($1)}'\'';echo \","$2","$1","$3","$4"\""}' | bash > ratings4.csv
```

Finally, just as before, put the transformed file in HDFS and run ImportTsv (much faster than the shell scripts)
```
HADOOP_USER_NAME=hdfs hdfs dfs -put ratings4.csv /tmp/
```
then create the table and import the data
```
echo create "'RATINGS4','rating'" | hbase shell
HADOOP_USER_NAME=hdfs  hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.separator=, -Dimporttsv.columns="HBASE_ROW_KEY,rating:movieid,rating:userid,rating:rating" RATINGS4 hdfs:///tmp/ratings4.csv
```
## Manipulating timestamps
By default, in shell operations or ImportTsv, because we've not specified a timestamp, the default system time is used. Since we do have a timestamp in our data, let's convince HBase to use it as the timestamp of the operation. We'll resort again to ImportTsv and specifically to a pre-defined variable called HBASE_TS_KEY. To be able to use it we will need the time to be expressed in miliseconds (it is in seconds in the file). We'll also define the ROWKEY as [movieid]:[userid]

```
tail -n +2 ratings_s.csv | awk -F, '{print $2":"$1","$2","$1","$3","$4"000"}' > ratings5.csv
HADOOP_USER_NAME=hdfs hdfs dfs -put ratings5.csv /tmp/
echo create "'RATINGS5','rating'" | hbase shell
HADOOP_USER_NAME=hdfs  hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.separator=, -Dimporttsv.columns="HBASE_ROW_KEY,rating:movieid,rating:userid,rating:rating,HBASE_TS_KEY" RATINGS5 hdfs:///tmp/ratings5.csv
```
now the data in "RATINGS5" is stored with the 'correct' timestamp

```
> scan "RATINGS5",{LIMIT=>3}
ROW                             COLUMN+CELL                                                                              
 1000:1733                      column=rating:movieid, timestamp=986711740000, value=1000                                
 1000:1733                      column=rating:rating, timestamp=986711740000, value=2.0                                  
 1000:1733                      column=rating:userid, timestamp=986711740000, value=1733                                 
 1000:2820                      column=rating:movieid, timestamp=986395650000, value=1000                                
 1000:2820                      column=rating:rating, timestamp=986395650000, value=3.0                                  
 1000:2820                      column=rating:userid, timestamp=986395650000, value=2820                                 
 1000:3032                      column=rating:movieid, timestamp=970350876000, value=1000                                
 1000:3032                      column=rating:rating, timestamp=970350876000, value=4.0                                  
 1000:3032                      column=rating:userid, timestamp=970350876000, value=3032                                 
3 row(s) in 0.1760 seconds
```
we could achieve a similar result by using an extra parameter to the ```put``` commands i.e. 
```
put 'table', 'row', 'cf:col', 'value', timestamp
```
