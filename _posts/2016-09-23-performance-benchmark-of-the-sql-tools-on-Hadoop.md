---
layout: post
title: "Performnace benchmark of the SQL tools on Hadoop"
---

就我手上現有的SQL on Hadoop工具

我做了一個簡單的benchmark去看效能

資料量不大，一個998177列、11欄的數據

(1欄字串、2欄double、7欄整數、一欄邏輯值)

存成json檔案(178MB), csv檔案(63MB)放置於hdfs

我們分別用HBase, Hive, Spark SQL, Drill去測試幾個簡單case


配備：

VM: 2-core with 4G ram x 3

版本：

|      | Hadoop | HBase | Hive  | Drill | Spark |
|------|--------|-------|-------|-------|-------|
| 版本 | 2.7.3  | 1.2.2 | 2.1.0 | 1.8.0 | 2.0.0 |

設備疲乏，可能只能給出一個簡單的比較

如果有更多機器，當然希望可以給出更多的比較

Hive可以使用mapreduce, tez或是Spark，儲存媒介也可以選hdfs或是hbase

但是這裡沒想測試那麼多，所以只測試mapreduce + hdfs的模式

而Spark就直接用Hive上的資料做Spark SQL

至於Drill則分別使用hdfs的json, csv檔案，以及透過HBase, Hive等方式去做


先稍微看一下資料：

``` bash
head tw5_df.csv
# nfbVD-5S-9.063,30,2015,1,1,0,0,5,TRUE,row1
# nfbVD-5N-9.013,11,2015,1,1,0,0,5,TRUE,row2
# nfbVD-5S-9.063,30,2015,1,1,0,1,5,TRUE,row3
# nfbVD-5N-9.013,11,2015,1,1,0,1,5,TRUE,row4
# nfbVD-5S-9.063,18,2015,1,1,0,2,5,TRUE,row5
# nfbVD-5N-9.013,5,2015,1,1,0,2,5,TRUE,row6
# nfbVD-5S-9.063,24,2015,1,1,0,3,5,TRUE,row7
# nfbVD-5N-9.013,6,2015,1,1,0,3,5,TRUE,row8
# nfbVD-5S-9.063,24,2015,1,1,0,4,5,TRUE,row9
# nfbVD-5N-9.013,6,2015,1,1,0,4,5,TRUE,row10

head tw5_df_hbase.csv
# nfbVD-5S-9.063,30,2015,1,1,0,0,5,TRUE,row0000001
# nfbVD-5N-9.013,11,2015,1,1,0,0,5,TRUE,row0000002
# nfbVD-5S-9.063,30,2015,1,1,0,1,5,TRUE,row0000003
# nfbVD-5N-9.013,11,2015,1,1,0,1,5,TRUE,row0000004
# nfbVD-5S-9.063,18,2015,1,1,0,2,5,TRUE,row0000005
# nfbVD-5N-9.013,5,2015,1,1,0,2,5,TRUE,row0000006
# nfbVD-5S-9.063,24,2015,1,1,0,3,5,TRUE,row0000007
# nfbVD-5N-9.013,6,2015,1,1,0,3,5,TRUE,row0000008
# nfbVD-5S-9.063,24,2015,1,1,0,4,5,TRUE,row0000009
# nfbVD-5N-9.013,6,2015,1,1,0,4,5,TRUE,row0000010

head tw5_df.json
# [
# {"vdid":"nfbVD-5S-9.063","volume":30,"date_year":2015,"date_month":1,"date_day":1,"time_hour":0,"time_minute":0,
# "weekday":5,"holiday":true,"speed":"84.26666667","laneoccupy":"12.13333333"},
# {"vdid":"nfbVD-5N-9.013","volume":11,"date_year":2015,"date_month":1,"date_day":1,"time_hour":0,"time_minute":0,
# "weekday":5,"holiday":true,"speed":"87.36363636","laneoccupy":"4.00000000"},
# {"vdid":"nfbVD-5S-9.063","volume":30,"date_year":2015,"date_month":1,"date_day":1,"time_hour":0,"time_minute":1,
# "weekday":5,"holiday":true,"speed":"84.26666667","laneoccupy":"12.13333333"},
# {"vdid":"nfbVD-5N-9.013","volume":11,"date_year":2015,"date_month":1,"date_day":1,"time_hour":0,"time_minute":1,
# "weekday":5,"holiday":true,"speed":"87.36363636","laneoccupy":"4.00000000"},
# {"vdid":"nfbVD-5S-9.063","volume":18,"date_year":2015,"date_month":1,"date_day":1,"time_hour":0,"time_minute":2,

# 放到hdfs上
hdfs dfs -mkdir /drill
hdfs dfs -put tw5_df.csv /drill/tw5_df.csv
hdfs dfs -put tw5_df.csv /drill/tw5_df_hive.csv
hdfs dfs -put tw5_df_hbase.csv /drill/tw5_df_hbase.csv
hdfs dfs -put tw5_df.json /drill/tw5_df.json
```


HBase使用hbase shell，然後用下面的script去input資料以及做query：

`hbase shell >`是打開hbase shell跑的意思，不然請在console跑

``` bash
## hbase shell > create 'vddata','vdid','vd_info','datetime'
hbase org.apache.hadoop.hbase.mapreduce.ImportTsv -Dimporttsv.columns="vdid,vd_info:volume,datetime:year,datetime:month,datetime:day,datetime:hour,datetime:minute,datetime:weekday,datetime:holiday,vd_info:speed,vd_info:laneoccupy,HBASE_ROW_KEY" '-Dimporttsv.separator=,' vddata /drill/tw5_df_hbase.csv

## hbase shell > scan 'vddata', {LIMIT => 1}
# ROW                                          COLUMN+CELL
#  row0000001                                  column=datetime:day, timestamp=1474478950775, value=2015
#  row0000001                                  column=datetime:holiday, timestamp=1474478950775, value=0
#  row0000001                                  column=datetime:hour, timestamp=1474478950775, value=1
#  row0000001                                  column=datetime:minute, timestamp=1474478950775, value=1
#  row0000001                                  column=datetime:month, timestamp=1474478950775, value=30
#  row0000001                                  column=datetime:weekday, timestamp=1474478950775, value=0
#  row0000001                                  column=datetime:year, timestamp=1474478950775, value=1.21333333333333E1
#  row0000001                                  column=vd_info:laneoccupy, timestamp=1474478950775, value=TRUE
#  row0000001                                  column=vd_info:speed, timestamp=1474478950775, value=5
#  row0000001                                  column=vd_info:volume, timestamp=1474478950775, value=8.42666666666667E1
#  row0000001                                  column=vdid:, timestamp=1474478950775, value=nfbVD-5S-9.063
# 1 row(s) in 0.0260 seconds

# 結果發現group by要用JAVA API去寫...就只好放棄了，留給Drill用
```


Hive則用下面的SQL script：

``` SQL
CREATE TABLE vddata (vdid STRING, speed DOUBLE, laneoccupy DOUBLE, volume INT, 
date_year INT, date_month INT, date_day INT, time_hour INT, time_minute INT, 
weekday INT, holiday BOOLEAN)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

LOAD DATA INPATH '/drill/tw5_df_hive.csv'
OVERWRITE INTO TABLE vddata;

select * from vddata limit 5;
# OK
# nfbVD-5S-9.063  30.0    2015    1       1       0       0       5       true
# nfbVD-5N-9.013  11.0    2015    1       1       0       0       5       true
# nfbVD-5S-9.063  30.0    2015    1       1       0       1       5       true
# nfbVD-5N-9.013  11.0    2015    1       1       0       1       5       true
# nfbVD-5S-9.063  18.0    2015    1       1       0       2       5       true
# Time taken: 0.86 seconds, Fetched: 5 row(s)

select count(vdid) from vddata;
# Total MapReduce CPU Time Spent: 4 seconds 590 msec
# OK
# Time taken: 27.214 seconds, Fetched: 1 row(s)

select date_month,count(vdid) from vddata group by date_month;
# Total MapReduce CPU Time Spent: 4 seconds 330 msec
# Time taken: 23.09 seconds, Fetched: 12 row(s)

select date_month,avg(speed),avg(laneoccupy),avg(volume) from vddata group by date_month;
# Total MapReduce CPU Time Spent: 4 seconds 100 msec
# Time taken: 19.655 seconds, Fetched: 12 row(s)

select date_month,date_day,avg(speed),avg(laneoccupy),avg(volume) from vddata group by date_month,date_day;
# Total MapReduce CPU Time Spent: 5 seconds 230 msec
# Time taken: 19.033 seconds, Fetched: 364 row(s)
```


再來是Spark SQL，要用Spark SQL就只能用

``` SQL
import org.apache.spark.sql.SparkSession
import java.util.Calendar._
import java.sql.Timestamp

val spark = SparkSession.builder().appName("spark on hive")
  .config("spark.sql.warehouse.dir", "hdfs://hc1/spark")
  .enableHiveSupport().getOrCreate()
 
val st = getInstance().getTime()
val sql_res_1 = spark.sql("select count(vdid) from vddata").collect()
println(getInstance().getTime().getTime() - st.getTime())
// 7724 ms

val st = getInstance().getTime()
val sql_res_2 = spark.sql("select date_month,count(vdid) from vddata group by date_month").collect()
println(getInstance().getTime().getTime() - st.getTime())
// 3729 ms

val st = getInstance().getTime()
val sql_res_3 = spark.sql("select date_month,avg(speed),avg(laneoccupy),avg(volume) from vddata group by date_month").collect()
println(getInstance().getTime().getTime() - st.getTime())
// 4520 ms

val st = getInstance().getTime()
val sql_res_4 = spark.sql("select date_month,date_day,avg(speed),avg(laneoccupy),avg(volume) from vddata group by date_month,date_day").collect()
println(getInstance().getTime().getTime() - st.getTime())
// 4353 ms
```

最後是Drill：

```
# on HBase
select count(vdid) from hbase.vddata;
# 1.319s
select vddata.datetime.`month`,count(vddata.vdid) from hbase.vddata group by vddata.datetime.`month`;
# 11.125s	
select vddata.datetime.`month`,avg(vddata.vd_info.volume) from hbase.vddata group by vddata.datetime.`month`;
# datatype error
select datetime.month,date_day,avg(vd_info.speed),avg(vd_info.laneoccupy),avg(vd_info.volume) 
from hbase.vddata group by datetime.month,datetime.day;
# datatype error

# on Hive
select count(vdid) from hive_cassSpark1.vddata;
# in time
select date_month,count(vdid) from hive_cassSpark1.vddata group by date_month;
# 2.203s	
select date_month,avg(speed),avg(laneoccupy),avg(volume) from hive_cassSpark1.vddata group by date_month;
# 2.632s
select date_month,date_day,avg(speed),avg(laneoccupy),avg(volume) 
from hive_cassSpark1.vddata group by date_month,date_day;
# 3.167s

# on csv in hdfs
select count(columns[0]) from dfs.`/drill/tw5_df.csv`;
# 1.078s
select columns[5],count(columns[0]) from dfs.`/drill/tw5_df.csv` group by columns[5];
# 2.049s
select columns[5],avg(columns[1]),avg(columns[2]),avg(columns[3]) from dfs.`/drill/tw5_df.csv` group by columns[5];
# datatype error 
select columns[5],columns[6],avg(columns[1]),avg(columns[2]),avg(columns[3]) 
from dfs.`/drill/tw5_df.csv` group by columns[5],columns[6];
# datatype error 

# on json in hdfs
select count(vdid) from dfs.`/drill/tw5_df.json`;
# 2.834s
select date_month,count(vdid) from dfs.`/drill/tw5_df.json` group by date_month;
# 5.778s	
select date_month,avg(speed),avg(laneoccupy),avg(volume) from dfs.`/drill/tw5_df.json` group by date_month;
# datatype error 
select date_month,date_day,avg(speed),avg(laneoccupy),avg(volume) from dfs.`/drill/tw5_df.json` group by date_month,date_day;
# datatype error 
```

datatype error是因為裡面有int, double混在同一列，Drill現在還無法有效處理這種問題

因此，透過Hive去做storage會是比較好的選擇

小結一下，在這樣的資料量(1M x 11)下，其實用Spark SQL沒有輸Drill太多

接下來測試看看大一點的資料量


我從UCI下載了兩個資料下來：[SUSY](http://archive.ics.uci.edu/ml/datasets/SUSY)跟[HIGGS](http://archive.ics.uci.edu/ml/datasets/HIGGS)

解壓縮後的大小分別是G跟G，然後直接用hive把資料放上去，分別用Spark SQL跟Drill去測試看看

先看一下資料長相

``` bash
head SUSY.csv
# 0.000000000000000000e+00,9.728614687919616699e-01,6.538545489311218262e-01,1.176224589347839355e+00,1.157156467437744141e+00,-1.739873170852661133e+00,-8.743090629577636719e-01,5.677649974822998047e-01,-1.750000417232513428e-01,8.100607395172119141e-01,-2.525521218776702881e-01,1.921887040138244629e+00,8.896374106407165527e-01,4.107718467712402344e-01,1.145620822906494141e+00,1.932632088661193848e+00,9.944640994071960449e-01,1.367815494537353516e+00,4.071449860930442810e-02
# 1.000000000000000000e+00,1.667973041534423828e+00,6.419061869382858276e-02,-1.225171446800231934e+00,5.061022043228149414e-01,-3.389389812946319580e-01,1.672542810440063477e+00,3.475464344024658203e+00,-1.219136357307434082e+00,1.295456290245056152e-02,3.775173664093017578e+00,1.045977115631103516e+00,5.680512785911560059e-01,4.819284379482269287e-01,0.000000000000000000e+00,4.484102725982666016e-01,2.053557634353637695e-01,1.321893453598022461e+00,3.775840103626251221e-01  
# 1.000000000000000000e+00,4.448399245738983154e-01,-1.342980116605758667e-01,-7.099716067314147949e-01,4.517189264297485352e-01,-1.613871216773986816e+00,-7.686609029769897461e-01,1.219918131828308105e+00,5.040258169174194336e-01,1.831247568130493164e+00,-4.313853085041046143e-01,5.262832045555114746e-01,9.415140151977539062e-01,1.587535023689270020e+00,2.024308204650878906e+00,6.034975647926330566e-01,1.562373995780944824e+00,1.135454416275024414e+00,1.809100061655044556e-01
# 1.000000000000000000e+00,3.812560737133026123e-01,-9.761453866958618164e-01,6.931523084640502930e-01,4.489588439464569092e-01,8.917528986930847168e-01,-6.773284673690795898e-01,2.033060073852539062e+00,1.533040523529052734e+00,3.046259880065917969e+00,-1.005284786224365234e+00,5.693860650062561035e-01,1.015211343765258789e+00,1.582216739654541016e+00,1.551914215087890625e+00,7.612152099609375000e-01,1.715463757514953613e+00,1.492256760597229004e+00,9.071890264749526978e-02 
# 1.000000000000000000e+00,1.309996485710144043e+00,-6.900894641876220703e-01,-6.762592792510986328e-01,1.589282631874084473e+00,-6.933256387710571289e-01,6.229069828987121582e-01,1.087561845779418945e+00,-3.817416727542877197e-01,5.892043709754943848e-01,1.365478992462158203e+00,1.179295063018798828e+00,9.682182073593139648e-01,7.285631299018859863e-01,0.000000000000000000e+00,1.083157896995544434e+00,4.342924803495407104e-02,1.154853701591491699e+00,9.485860168933868408e-02

head HIGGS.csv
# 1.000000000000000000e+00,8.692932128906250000e-01,-6.350818276405334473e-01,2.256902605295181274e-01,3.274700641632080078e-01,-6.899932026863098145e-01,7.542022466659545898e-01,-2.485731393098831177e-01,-1.092063903808593750e+00,0.000000000000000000e+00,1.374992132186889648e+00,-6.536741852760314941e-01,9.303491115570068359e-01,1.107436060905456543e+00,1.138904333114624023e+00,-1.578198313713073730e+00,-1.046985387802124023e+00,0.000000000000000000e+00,6.579295396804809570e-01,-1.045456994324922562e-02,-4.576716944575309753e-02,3.101961374282836914e+00,1.353760004043579102e+00,9.795631170272827148e-01,9.780761599540710449e-01,9.200048446655273438e-01,7.216574549674987793e-01,9.887509346008300781e-01,8.766783475875854492e-01"
# 1.000000000000000000e+00,9.075421094894409180e-01,3.291472792625427246e-01,3.594118654727935791e-01,1.497969865798950195e+00,-3.130095303058624268e-01,1.095530629158020020e+00,-5.575249195098876953e-01,-1.588229775428771973e+00,2.173076152801513672e+00,8.125811815261840820e-01,-2.136419266462326050e-01,1.271014571189880371e+00,2.214872121810913086e+00,4.999939501285552979e-01,-1.261431813240051270e+00,7.321561574935913086e-01,0.000000000000000000e+00,3.987008929252624512e-01,-1.138930082321166992e+00,-8.191101951524615288e-04,0.000000000000000000e+00,3.022198975086212158e-01,8.330481648445129395e-01,9.856996536254882812e-01,9.780983924865722656e-01,7.797321677207946777e-01,9.923557639122009277e-01,7.983425855636596680e-01"  
# 1.000000000000000000e+00,7.988347411155700684e-01,1.470638751983642578e+00,-1.635974764823913574e+00,4.537731707096099854e-01,4.256291687488555908e-01,1.104874610900878906e+00,1.282322287559509277e+00,1.381664276123046875e+00,0.000000000000000000e+00,8.517372012138366699e-01,1.540658950805664062e+00,-8.196895122528076172e-01,2.214872121810913086e+00,9.934899210929870605e-01,3.560801148414611816e-01,-2.087775468826293945e-01,2.548224449157714844e+00,1.256954550743103027e+00,1.128847599029541016e+00,9.004608392715454102e-01,0.000000000000000000e+00,9.097532629966735840e-01,1.108330488204956055e+00,9.856922030448913574e-01,9.513312578201293945e-01,8.032515048980712891e-01,8.659244179725646973e-01,7.801175713539123535e-01"      
# 0.000000000000000000e+00,1.344384789466857910e+00,-8.766260147094726562e-01,9.359127283096313477e-01,1.992050051689147949e+00,8.824543952941894531e-01,1.786065936088562012e+00,-1.646777749061584473e+00,-9.423825144767761230e-01,0.000000000000000000e+00,2.423264741897583008e+00,-6.760157942771911621e-01,7.361586689949035645e-01,2.214872121810913086e+00,1.298719763755798340e+00,-1.430738091468811035e+00,-3.646581768989562988e-01,0.000000000000000000e+00,7.453126907348632812e-01,-6.783788204193115234e-01,-1.360356330871582031e+00,0.000000000000000000e+00,9.466524720191955566e-01,1.028703689575195312e+00,9.986560940742492676e-01,7.282806038856506348e-01,8.692002296447753906e-01,1.026736497879028320e+00,9.579039812088012695e-01" 
# 1.000000000000000000e+00,1.105008959770202637e+00,3.213555514812469482e-01,1.522401213645935059e+00,8.828076124191284180e-01,-1.205349326133728027e+00,6.814661026000976562e-01,-1.070463895797729492e+00,-9.218706488609313965e-01,0.000000000000000000e+00,8.008721470832824707e-01,1.020974040031433105e+00,9.714065194129943848e-01,2.214872121810913086e+00,5.967612862586975098e-01,-3.502728641033172607e-01,6.311942934989929199e-01,0.000000000000000000e+00,4.799988865852355957e-01,-3.735655248165130615e-01,1.130406111478805542e-01,0.000000000000000000e+00,7.558564543724060059e-01,1.361057043075561523e+00,9.866096973419189453e-01,8.380846381187438965e-01,1.133295178413391113e+00,8.722448945045471191e-01,8.084865212440490723e-01
```

``` bash
hdfs dfs -put SUSY.csv.gz /drill/SUSY.csv.gz
hdfs dfs -put HIGGS.csv.gz /drill/HIGGS.csv.gz
```

Hive上傳local file

``` SQL
CREATE TABLE susy_df (lepton1pt DOUBLE, lepton1eta DOUBLE, lepton1phi DOUBLE, 
lepton2pt DOUBLE, lepton2eta DOUBLE, lepton2phi DOUBLE, mem DOUBLE, mep DOUBLE, 
met_rel DOUBLE, axialmet DOUBLE, m_r DOUBLE, m_tr_2 DOUBLE, r DOUBLE, mt2 DOUBLE,
s_r DOUBLE, m_delta_r DOUBLE, dphi_r_b DOUBLE, cos_theta_r1 DOUBLE)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

LOAD DATA LOCAL INPATH '/home/tester/SUSY.csv.gz'
OVERWRITE INTO TABLE susy_df;
-- Time taken: 19.125 seconds

CREATE TABLE higgs_df (lepton_pt DOUBLE, lepton_eta DOUBLE, lepton_phi DOUBLE,
mem DOUBLE, mep DOUBLE, jet1pt DOUBLE, jet1eta DOUBLE, jet1phi DOUBLE, 
jet1b_tag DOUBLE, jet2pt DOUBLE, jet2eta DOUBLE, jet2phi DOUBLE, jet2b_tag DOUBLE,
jet3pt DOUBLE, jet3eta DOUBLE, jet3phi DOUBLE, jet3b_tag DOUBLE, jet4pt DOUBLE,
jet4eta DOUBLE, jet4phi DOUBLE, jet4b_tag DOUBLE, m_jj DOUBLE, m_jjj DOUBLE,
m_lv DOUBLE, m_jlv DOUBLE, m_bb DOUBLE, m_wbb DOUBLE, m_wwbb DOUBLE)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

LOAD DATA LOCAL INPATH '/home/tester/HIGGS.csv.gz'
OVERWRITE INTO TABLE higgs_df;
-- Time taken: 50.145 seconds
```

複雜一點的運算，或是filter，我的VM都撐不住，就只能等到有好電腦才有機會測試了

這裡就簡單測試aggregation

`select count(lepton1phi) from susy_df group by lepton1pt`
`select count(lepton_phi) from higgs_df group by lepton_pt`

|       |   Hive   |   Spark  |   Drill  |
|-------|----------|----------|----------|
| SUSY  |  44.374s |  28.794s |  29.262s |
| HIGGS | 105.887s |  58.748s |      74s |

Spark script:

``` scala
import org.apache.spark.sql.SparkSession
import java.util.Calendar._
import java.sql.Timestamp

val spark = SparkSession.builder().appName("spark on hive").config("spark.sql.warehouse.dir", "hdfs://hc1/spark").enableHiveSupport().getOrCreate()

val st = getInstance().getTime()
val susy_out_3 = spark.sql("select count(lepton1phi) from susy_df group by lepton1pt").collect()
println(getInstance().getTime().getTime() - st.getTime())

val st = getInstance().getTime()
val tmp = spark.sql("select count(lepton_phi) from higgs_df group by lepton_pt").collect()
println(getInstance().getTime().getTime() - st.getTime())

```

Drill運行SQL script

``` SQL
select count(lepton1phi) from hive_cassSpark1.susy_df group by lepton1pt
select count(lepton_phi) from hive_cassSpark1.higgs_df group by lepton_pt
```

在這麼大量的資料中，Spark用了三個核心，Drill只用了一個核心

這樣的差距都還在可以接受的範圍，Drill稍微微調後，應該可以在相同的電腦設備下

擁有比Spark SQL還強大的運算能力


結論，Drill在我看來會是比較長久的發展，只是它只適合用來做為讀取介面

