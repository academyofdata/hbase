# HBase Metadata Management
to see metadata informations about a specific table (say "RATINGS2") one has to use hbase:meta (or .META. in older HBase versions) table
```
scan 'hbase:meta',{ROWPREFIXFILTER=>'RATINGS2'}
```
this will output the current information about the table "RATINGS2" (actually all tables starting with "RATINGS2") and will look something like
```
 RATINGS2,,1543080471147.ccaa039 column=info:regioninfo, timestamp=1543080471536, value={ENCODED => ccaa039a830618aff539c432
 a830618aff539c432f62b3029.      f62b3029, NAME => 'RATINGS2,,1543080471147.ccaa039a830618aff539c432f62b3029.', STARTKEY => 
                                 '', ENDKEY => ''}                                                                          
 RATINGS2,,1543080471147.ccaa039 column=info:seqnumDuringOpen, timestamp=1543080471696, value=\x00\x00\x00\x00\x00\x00\x00\x
 a830618aff539c432f62b3029.      02                                                                                         
 RATINGS2,,1543080471147.ccaa039 column=info:server, timestamp=1543080471696, value=instance-53720.bigstep.io:16020         
 a830618aff539c432f62b3029.
```
for each region in the table there will be four columns (part of the "info" CF) returned
* info:server - with information about the Region Server
* info:seqnumDuringOpen - 
* info:regioninfo - with information about the region
* info:serverstartcode

In the case above there's only one region, so only 4 columns returned.
Let's now split this single region into two, starting at key '2'

```
split 'RATINGS2','2'
```

if we do now again the scan we'll get something similar with

```
> scan 'hbase:meta',{ROWPREFIXFILTER=>'RATINGS2'}
ROW                              COLUMN+CELL                                                                                
 RATINGS2,,1543148749479.fb1ca2a column=info:regioninfo, timestamp=1543148749725, value={ENCODED => fb1ca2a7c9fcda82049aaf02
 7c9fcda82049aaf02af142edd.      af142edd, NAME => 'RATINGS2,,1543148749479.fb1ca2a7c9fcda82049aaf02af142edd.', STARTKEY => 
                                 '', ENDKEY => '2'}                                                                         
 RATINGS2,,1543148749479.fb1ca2a column=info:seqnumDuringOpen, timestamp=1543148749762, value=\x00\x00\x00\x00\x00=\x0CO    
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,,1543148749479.fb1ca2a column=info:server, timestamp=1543148749762, value=instance-53720.bigstep.io:16020         
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,,1543148749479.fb1ca2a column=info:serverstartcode, timestamp=1543148749762, value=1543061757407                  
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,2,1543148749479.fceee9 column=info:regioninfo, timestamp=1543148749725, value={ENCODED => fceee93c06466f7bd4ebcaa6
 3c06466f7bd4ebcaa6889a49a2.     889a49a2, NAME => 'RATINGS2,2,1543148749479.fceee93c06466f7bd4ebcaa6889a49a2.', STARTKEY =>
                                  '2', ENDKEY => ''}                                                                        
 RATINGS2,2,1543148749479.fceee9 column=info:seqnumDuringOpen, timestamp=1543148749758, value=\x00\x00\x00\x00\x00=\x0CP    
 3c06466f7bd4ebcaa6889a49a2.                                                                                                
 RATINGS2,2,1543148749479.fceee9 column=info:server, timestamp=1543148749758, value=instance-53720.bigstep.io:16020         
 3c06466f7bd4ebcaa6889a49a2.                                                                                                
 RATINGS2,2,1543148749479.fceee9 column=info:serverstartcode, timestamp=1543148749758, value=1543061757407                  
 3c06466f7bd4ebcaa6889a49a2.        
```
we can already see that there are twice as many results as before as there are two regions reported.
Let's further split the newly created region into two. For this we'll have to pay attention to the info:regioninfo column, the "NAME" field - which represents the region name, we'll need it to tell HBase that we want now the region to be split, not the whole table
```
> split 'RATINGS2,2,1543148749479.fceee93c06466f7bd4ebcaa6889a49a2.','4'

> scan 'hbase:meta',{ROWPREFIXFILTER=>'RATINGS2'}
ROW                              COLUMN+CELL                                                                                
 RATINGS2,,1543148749479.fb1ca2a column=info:regioninfo, timestamp=1543148749725, value={ENCODED => fb1ca2a7c9fcda82049aaf02
 7c9fcda82049aaf02af142edd.      af142edd, NAME => 'RATINGS2,,1543148749479.fb1ca2a7c9fcda82049aaf02af142edd.', STARTKEY => 
                                 '', ENDKEY => '2'}                                                                         
 RATINGS2,,1543148749479.fb1ca2a column=info:seqnumDuringOpen, timestamp=1543148749762, value=\x00\x00\x00\x00\x00=\x0CO    
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,,1543148749479.fb1ca2a column=info:server, timestamp=1543148749762, value=instance-53720.bigstep.io:16020         
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,,1543148749479.fb1ca2a column=info:serverstartcode, timestamp=1543148749762, value=1543061757407                  
 7c9fcda82049aaf02af142edd.                                                                                                 
 RATINGS2,2,1543148749479.fceee9 column=info:regioninfo, timestamp=1543148894516, value={ENCODED => fceee93c06466f7bd4ebcaa6
 3c06466f7bd4ebcaa6889a49a2.     889a49a2, NAME => 'RATINGS2,2,1543148749479.fceee93c06466f7bd4ebcaa6889a49a2.', STARTKEY =>
                                  '2', ENDKEY => '', OFFLINE => true, SPLIT => true}                                        
 RATINGS2,2,1543148749479.fceee9 column=info:seqnumDuringOpen, timestamp=1543148749758, value=\x00\x00\x00\x00\x00=\x0CP    
 3c06466f7bd4ebcaa6889a49a2.                                                                                                
 RATINGS2,2,1543148749479.fceee9 column=info:server, timestamp=1543148749758, value=instance-53720.bigstep.io:16020         
 3c06466f7bd4ebcaa6889a49a2.                                                                                                
 RATINGS2,2,1543148749479.fceee9 column=info:serverstartcode, timestamp=1543148749758, value=1543061757407                  
 3c06466f7bd4ebcaa6889a49a2.                                                                                                
 RATINGS2,2,1543148749479.fceee9 column=info:splitA, timestamp=1543148894516, value={ENCODED => 66d912622f2e17207895846df348
 3c06466f7bd4ebcaa6889a49a2.     0216, NAME => 'RATINGS2,2,1543148894312.66d912622f2e17207895846df3480216.', STARTKEY => '2'
                                 , ENDKEY => '4'}                                                                           
 RATINGS2,2,1543148749479.fceee9 column=info:splitB, timestamp=1543148894516, value={ENCODED => 3e347ac3933edc46a29a7bef7493
 3c06466f7bd4ebcaa6889a49a2.     5322, NAME => 'RATINGS2,4,1543148894312.3e347ac3933edc46a29a7bef74935322.', STARTKEY => '4'
                                 , ENDKEY => ''}                                                                            
 RATINGS2,2,1543148894312.66d912 column=info:regioninfo, timestamp=1543148894516, value={ENCODED => 66d912622f2e17207895846d
 622f2e17207895846df3480216.     f3480216, NAME => 'RATINGS2,2,1543148894312.66d912622f2e17207895846df3480216.', STARTKEY =>
                                  '2', ENDKEY => '4'}                                                                       
 RATINGS2,2,1543148894312.66d912 column=info:seqnumDuringOpen, timestamp=1543148894543, value=\x00\x00\x00\x00\x00=\x0CP    
 622f2e17207895846df3480216.            
  RATINGS2,2,1543148894312.66d912 column=info:server, timestamp=1543148894543, value=instance-53720.bigstep.io:16020         
 622f2e17207895846df3480216.                                                                                                
 RATINGS2,2,1543148894312.66d912 column=info:serverstartcode, timestamp=1543148894543, value=1543061757407                  
 622f2e17207895846df3480216.                                                                                                
 RATINGS2,4,1543148894312.3e347a column=info:regioninfo, timestamp=1543148894516, value={ENCODED => 3e347ac3933edc46a29a7bef
 c3933edc46a29a7bef74935322.     74935322, NAME => 'RATINGS2,4,1543148894312.3e347ac3933edc46a29a7bef74935322.', STARTKEY =>
                                  '4', ENDKEY => ''}                                                                        
 RATINGS2,4,1543148894312.3e347a column=info:seqnumDuringOpen, timestamp=1543148894539, value=\x00\x00\x00\x00\x00=\x0CQ    
 c3933edc46a29a7bef74935322.                                                                                                
 RATINGS2,4,1543148894312.3e347a column=info:server, timestamp=1543148894539, value=instance-53720.bigstep.io:16020         
 c3933edc46a29a7bef74935322.                                                                                                
 RATINGS2,4,1543148894312.3e347a column=info:serverstartcode, timestamp=1543148894539, value=1543061757407                  
 c3933edc46a29a7bef74935322.    

```



