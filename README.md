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
