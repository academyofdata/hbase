# Apache HBase basics
## Preparation
start by downloading the data files
```
export OUTDIR=.
wget https://raw.githubusercontent.com/academyofdata/data/master/movies-with-year.csv -O $OUTDIR/movies.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings2.csv -O $OUTDIR/ratings_s.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings.csv.gz -O $OUTDIR/ratings.csv.gz
gunzip -f $OUTDIR/ratings.csv.gz
wget https://raw.githubusercontent.com/academyofdata/data/master/users.csv -O $OUTDIR/users.csv
```
## ImportTsv from denormalized ratings file
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

we could now put the files into HDFS
```
HADOOP_USER_NAME=hdfs hdfs dfs -put $OUTDIR/ratings1.csv /tmp/
```
before loading the data, let's create the HBase table, so use the shell to do it
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

## Importing with an HBase shell script
Let's now use the simplified ratings file as an imput to transform and load into a separate RATINGS2 table. The RATINGS2 has a single colum family (rating) but each entry has a different column name based on the user that rates the current movie (say 4321), that is the user id is being made part of the column name; so you have rating:rating1234 with value 4.0 to signal that the user 1234 has rated movie 4321 with a rating of 4

```
echo "create 'RATINGS2','rating'" > ratings2.txt
tail -n +2 ratings_s.csv | awk -v tbl=RATINGS2 -F, '{print "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:rating" $1 "'\'','\''" $3 "'\''\n" "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:timestamp" $1 "'\'','\''" $4 "'\''"}' >> ratings2.txt
```
now we have a script that we could pass to hbase shell for execution
```
hbase shell ./ratings2.txt
```
(this will take a while to execute; for a "non-blocking" variant try ```hbase shell  ./ratings2.txt > shell.out 2>&1 &```)

let's now use the same technique, but let's limit ourselves to storing just two pieces of information for each rating - the userid and the rating. We will thus use a column that will be called e.g. 310 for the user with the id 310, the value in that column will be the rating of the said user.

```
echo "create 'RATINGS3','rating'" > ratings3.txt
tail -n +2 ratings_s.csv | awk -v tbl=RATINGS3 -F, '{print "put '\''" tbl "'\'','\''" $2 "'\'','\''rating:" $1 "'\'','\''" $3 "'\''"}' >> ratings3.txt
```
then use a shell to load it into HBase
```
hbase shell ./ratings3.txt
```

## ROWKEY management
Let's assume we'll want to lookup these ratings by the time they were given, rather than by movieid (as it was the case so far). This means we should include the timestamp as the first part of the ROWKEY. The simplest way would be to transform the input file using the following line, then using ImportTsv to load it 
```
tail -n +2 ratings_s.csv | awk -F, '{print $4":"$2","$1","$3}' > ratings4.csv
```


