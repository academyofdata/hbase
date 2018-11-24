# hbase
start by downloading the data files
```
export OUTDIR=.
wget https://raw.githubusercontent.com/academyofdata/data/master/movies-with-year.csv -O $OUTDIR/movies.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings2.csv -O $OUTDIR/ratings_s.csv
wget https://raw.githubusercontent.com/academyofdata/inputs/master/ratings.csv.gz -O $OUTDIR/ratings.csv.gz
gunzip -f $OUTDIR/ratings.csv.gz
wget https://raw.githubusercontent.com/academyofdata/data/master/users.csv -O $OUTDIR/users.csv
```
use bash and some utilities to prepare files in a format that's suitable for importtsv 
```
awk -F, -v u=1 -v m=8 -v OFS="," -v ORS="" '{print $m":"$u",";for(i=1;i<=NF;i++)printf("%s%s",$i,(i!=NF)?OFS:"\n")}' $OUTDIR/ratings.csv > $OUTDIR/ratings1.csv

```
put the files into HDFS
```
HADOOP_USER_NAME=hdfs hdfs dfs -put $OUTDIR/ratings1.csv /tmp/
```

