03当作手机日志的客户端读取/usr/local/src/zebra/data/本地文件为数据源，01、02为两服务器做负载均衡

启动：

../bin/flume-ng agent -c ./ -f ./flume-zebra.properties -n a1 -Dflume.root.logger=INFO,console

报错可以修改修改java最大内存大小
vi bin/flume-ng
JAVA_OPTS="-Xmx1024m"

数据清洗：
创建数据库：

create database zebra;
use zebra;
建立表：

create EXTERNAL table zebra (a1 string,a2 string,a3 string,a4 string,a5 string,a6 string,a7 string,a8
string,a9 string,a10 string,a11 string,a12 string,a13 string,a14 string,a15 string,a16 string,a17
string,a18 string,a19 string,a20 string,a21 string,a22 string,a23 string,a24 string,a25 string,a26
string,a27 string,a28 string,a29 string,a30 string,a31 string,a32 string,a33 string,a34 string,a35
string,a36 string,a37 string,a38 string,a39 string,a40 string,a41 string,a42 string,a43 string,a44
string,a45 string,a46 string,a47 string,a48 string,a49 string,a50 string,a51 string,a52 string,a53
string,a54 string,a55 string,a56 string,a57 string,a58 string,a59 string,a60 string,a61 string,a62
string,a63 string,a64 string,a65 string,a66 string,a67 string,a68 string,a69 string,a70 string,a71
string,a72 string,a73 string,a74 string,a75 string,a76 string,a77 string) partitioned by (reportTime
string) row format delimited fields terminated by '|' stored as textfile location '/zebra';

ALTER TABLE zebra add PARTITION (reportTime='2019-03-31') location '/zebra/reportTime=2019-03-31';

alter table zebra drop partition (reportTime='2019-03-31');

select * from zebra TABLESAMPLE (1 ROWS);

开始清洗数据从原来的77个字段变为23个字段：
create table dataclear(reporttime string,appType bigint,appSubtype bigint,userIp string,userPort
bigint,appServerIP string,appServerPort bigint,host string,cellid string,appTypeCode
bigint,interruptType String,transStatus bigint,trafficUL bigint,trafficDL bigint,retranUL
bigint,retranDL bigint,procdureStartTime bigint,procdureEndTime bigint)row format delimited fields
terminated by '|';

从zebra表里导出数据到dataclear表里（23个字段的值）

insert into table dataclear select reporttime,a23,a24,a27,a29,a31,a33,a59,if(a17=="","0000",a17),a19,a68,a55,a34,a35,a40,a41,a20,a21 from zebra where reportTime='2019-03-31';

处理数据：
create table dataproc (reporttime string,appType bigint,appSubtype bigint,userIp string,userPort
bigint,appServerIP string,appServerPort bigint,host string,cellid string,attempts bigint,accepts
bigint,trafficUL bigint,trafficDL bigint,retranUL bigint,retranDL bigint,failCount bigint,transDelay
bigint)row format delimited fields terminated by '|';

处理业务逻辑，得到dataproc表
create table dataproc (reporttime string,appType bigint,appSubtype bigint,userIp string,userPort
bigint,appServerIP string,appServerPort bigint,host string,cellid string,attempts bigint,accepts
bigint,trafficUL bigint,trafficDL bigint,retranUL bigint,retranDL bigint,failCount bigint,transDelay
bigint)row format delimited fields terminated by '|';

根据业务规则，做字段处理
建表语句：
insert overwrite table dataproc select
reporttime,appType,appSubtype,userIp,userPort,appServerIP,appServerPort,host,
if(cellid == '',"000000000",cellid),if(appTypeCode == 103,1,0),if(appTypeCode == 103 and
find_in_set(transStatus,"10,11,12,13,14,15,32,33,34,35,36,37,38,48,49,50,51,52,53,54,55,199,200,201,2
02,203,204,205,206,302,304,306")!=0 and interruptType == 0,1,0),if(apptypeCode == 103,trafficUL,0),
if(apptypeCode == 103,trafficDL,0), if(apptypeCode == 103,retranUL,0), if(apptypeCode ==
103,retranDL,0), if(appTypeCode == 103 and transStatus == 1 and interruptType ==
0,1,0),if(appTypeCode == 103, procdureEndTime - procdureStartTime,0) from dataclear;


查询关心的信息，以应用受欢迎程度表为例：
建表语句：
create table D_H_HTTP_APPTYPE(hourid string,appType int,appSubtype int,attempts bigint,accepts
bigint,succRatio double,trafficUL bigint,trafficDL bigint,totalTraffic bigint,retranUL
bigint,retranDL bigint,retranTraffic bigint,failCount bigint,transDelay bigint) row format delimited
fields terminated by '|';

根据总表dataproc,按条件做聚合以及字段的累加
建表语句：

insert overwrite table D_H_HTTP_APPTYPE select reporttime,apptype,appsubtype,sum(attempts),sum(accepts),round(sum(accepts)/sum(attempts),2),sum(trafficUL),sum(trafficDL),sum(trafficUL)+sum(trafficDL),sum(retranUL),sum(retranDL),sum(retranUL)+sum(retranDL),sum(failCount),sum(transDelay)from dataproc group by reporttime,apptype,appsubtype;

查询前5名受欢迎app
select hourid,apptype,sum(totalTraffic) as tt from D_H_HTTP_APPTYPE group by hourid,apptype sort by tt desc limit 5;

用sqoop导出到mysql：

创建数据库：
创建表：
create table D_H_HTTP_APPTYPE(hourid varchar(250),appType int,appSubtype int,attempts bigint,accepts
bigint,succRatio double,trafficUL bigint,trafficDL bigint,totalTraffic bigint,retranUL
bigint,retranDL bigint,retranTraffic bigint,failCount bigint,transDelay bigint);
导入到hdfs语句：
./sqoop import --connect jdbc:mysql://hadoop01:3306/mydb --username root --password 123456 --target-dir '/user/hive/warehouse/zebra.db/d_h_http_apptype/000000_0' --table D_H_HTTP_APPTYPE -m 1 --fields-terminated-by '|';
强制指定一亿个map切分：
-m 1
导出到mysql
 ./sqoop export --connect jdbc:mysql://hadoop01:3306/mydb --username root --password 123456 --export-dir '/user/hive/warehouse/zebra.db/d_h_http_apptype' --table D_H_HTTP_APPTYPE -m 1 --fields-terminated-by '|'


