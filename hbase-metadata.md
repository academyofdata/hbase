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

