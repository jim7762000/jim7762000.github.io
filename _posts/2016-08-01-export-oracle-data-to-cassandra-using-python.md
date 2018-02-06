---
layout: post
title: "用Python將Oracle DB的資料匯出到Cassandra"
---

基本上的工具在前幾篇都安裝完

這一篇主要目的是把Oracle資料庫的資料匯出到Cassandra中

原本這部分要用sqoop做，結果發現沒辦法使用

只好用Python自己做輪子，於是這篇就誕生了

1. 準備工作

要有Oracle DB, Cassandra cluster跟安裝好Python本機

我的架構是：

5台VM，每一台都是4G RAM，兩核心

1台是Oracle DB，其他四台是Cassandra Cluster

本機是四核心八執行緒，32GB的電腦，所以全部電腦在跑，本機會有點喘XDD

2. Python程式

上傳、刪除Oracle DB的部分就參考cx_Oracle那篇

這篇程式主要是做了一些type mapping、cassandra上傳加速的測試

最後發現使用`execute_concurrent_with_args`可以比正常用for + execute快上7倍

``` python
#!/usr/bin/python

import sys, cx_Oracle, datetime, time
import cassandra.cluster
import cassandra.concurrent
from itertools import islice

class TooManyArgsError(Exception):
    """Err type for too many arguments."""
    pass

# import sys, getopt

def main(argv):
    # information for connecting to oracle
    oracleLoginInfo = u'system/qscf12356@192.168.0.120:1521/orcl'
    # define the number of fetching once = the number of rows uploaded to cassandra
    numFetch = 5000

    # cassandra cluster ips
    cassandraIpAddrs = [u'192.168.0.161', u'192.168.0.162', \
                        u'192.168.0.163', u'192.168.0.164'];
    # create a dict to map the data type of python to the one in Cassandra
    pythonCassTypeMap = dict([["str", "text"], ["int", "int"], \
        ["float", "double"], ["datetime", "timestamp"]])

    #%% input check
    if len(argv) is 0:
        tableListFN = 'dataTable.csv'
        print """The default filename of the table list is '%s'.""" % tableListFN
    elif len(argv) > 1:
        raise TooManyArgsError('Too many input arguments!')
    else:
        tableListFN = argv[0]

    #%% read input file
    # print log
    print 'The filename you input is %s.' % tableListFN
    # read the title row of csv
    fieldnames = None
    with open(tableListFN) as csvfile:
        firstRow = csvfile.readlines(1)
        fieldnames = tuple(firstRow[0].strip('\n').split(","))

    # read the rows of csv
    tableList = list()
    with open(tableListFN) as csvfile:
        for row in islice(csvfile, 1, None):
            values = [elem.upper() for elem in row.strip('\n').split(",")]
            tableList.append(dict(zip(fieldnames, values)))

    if len(tableList) is 0:
        print 'There is no data in %s.' % tableListFN

    for row in tableList:
        print 'Now is upload the dataset %s.%s...' % \
            (row['oracleSchema'].lower(), row['oracleTblName'].lower())
        #%% import data from Oracle database
        # create connection
        oracleConn = cx_Oracle.connect(oracleLoginInfo)
        # activate cursor
        oracleCursor = oracleConn.cursor()
        # make sql
        oracleSql = "select rowid,t.* from %s.%s t" % \
            (row['oracleSchema'], row['oracleTblName'])
        # execute sql query
        oracleCursor.execute(oracleSql)

        #%% start to fetch rows and upload to cassandra
        # connect to cassandra
        casCon = cassandra.cluster.Cluster(cassandraIpAddrs)
        casSession = casCon.connect()

        # create keyspace if it does not exist
        casSession.execute("""CREATE KEYSPACE IF NOT EXISTS %s
            WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '2'}
            """ % row['cassandraKeyspace'])
        # set to keyspace
        casSession.set_keyspace(row['cassandraKeyspace'].lower())

        #%% get the oracle data and check the data type
        oracleRows = oracleCursor.fetchmany(numFetch)
        # get data type
        colDataType = [type(r).__name__ for r in oracleRows[0]]
        # find the mapping data type in cassandra
        cassType = [pythonCassTypeMap[dataType] for dataType in colDataType]
        # acquire the column names in Oracle DB
        oracleColNames = [x[0].lower() for x in oracleCursor.description]

        # create table  if it does not exist
        casSession.execute('''CREATE TABLE IF NOT EXISTS {}.{} ({}, PRIMARY KEY (rowid))'''.format(\
            row['cassandraKeyspace'].lower(), row['cassandraTblName'].lower(),\
                ','.join([x + ' ' + y for (x,y) in zip(oracleColNames, cassType)])))

        #%% upload data to cassandra
        insertCQL = """INSERT INTO {}.{} ({}) VALUES ({})""".format(\
                    row['cassandraKeyspace'].lower(), row['cassandraTblName'].lower(), \
                    ','.join(oracleColNames), \
                    ','.join(['%s' for x in range(len(oracleColNames))]))
        preparedCQL = casSession.prepare(insertCQL.replace('%s', '?'))

        print 'start upload at %s...' % datetime.datetime.now()
        st = time.clock()

        numRowsOracle = len(oracleRows)
        while len(oracleRows) is not 0:
            ## first 750,000 cells ~ 55 secs
            # for dataRow in oracleRows:
                # casSession.execute(insertCQL, dataRow)
            ## second 750,000 cells ~ 50 secs
            # for dataRow in oracleRows:
                # casSession.execute(preparedCQL, dataRow)
            ## third 750,000 cells ~ 15 secs
            # for dataRow in oracleRows:
                # casSession.execute_async(preparedCQL, dataRow)
            ## fourth 750,000 cells ~ 8 secs
            oracleRows = oracleCursor.fetchmany(numFetch)
            cassandra.concurrent.execute_concurrent_with_args(casSession, preparedCQL, oracleRows, concurrency = 50)
            numRowsOracle += len(oracleRows)                   

        print 'End upload at %s...' % datetime.datetime.now()
        print 'The total upload time for %i cells is %s seconds...\n' % \
            (numRowsOracle * len(oracleColNames), time.clock() - st)

        # shutdown the connection to cassandra
        casCon.shutdown()
        # clost the connection to Oracle
        oracleCursor.close()
        oracleConn.close()

if __name__ == "__main__":
    main(sys.argv[1:])
```

之後又寫了一版同時用pp去平行的處理各張表格

手上表格不多(六個)，反正處理時間大概就是最大那張表格的時間：

``` python
#!/usr/bin/python

from itertools import islice
import sys, pp

class TooManyArgsError(Exception):
    """Err type for too many arguments."""
    pass

def implementUploadTbl(row):
    import cx_Oracle
    import cassandra.cluster
    import cassandra.concurrent

    # information for connecting to oracle
    oracleLoginInfo = u'system/qscf12356@192.168.0.120:1521/orcl'
    # define the number of fetching once = the number of rows uploaded to cassandra
    numFetch = 5000

    # cassandra cluster ips
    cassandraIpAddrs = [u'192.168.0.161', u'192.168.0.162', \
                        u'192.168.0.163', u'192.168.0.164'];    
    
    # create a dict to map the data type of python to the one in Cassandra
    pythonCassTypeMap = dict([["str", "text"], ["int", "int"], \
        ["float", "double"], ["datetime", "timestamp"]])

    #%% import data from Oracle database
    # create connection
    oracleConn = cx_Oracle.connect(oracleLoginInfo)
    # activate cursor
    oracleCursor = oracleConn.cursor()
    # make sql
    oracleSql = "select rowid,t.* from %s.%s t" % \
        (row['oracleSchema'], row['oracleTblName'])
    # execute sql query
    oracleCursor.execute(oracleSql)

    #%% start to fetch rows and upload to cassandra
    # connect to cassandra
    casCon = cassandra.cluster.Cluster(cassandraIpAddrs)
    casSession = casCon.connect()

    # create keyspace if it does not exist
    casSession.execute("""CREATE KEYSPACE IF NOT EXISTS %s
        WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '2'}
        """ % row['cassandraKeyspace'])
    # set to keyspace
    casSession.set_keyspace(row['cassandraKeyspace'].lower())

    #%% get the oracle data and check the data type
    oracleRows = oracleCursor.fetchmany(numFetch)
    # get data type
    colDataType = [type(r).__name__ for r in oracleRows[0]]
    # find the mapping data type in cassandra
    cassType = [pythonCassTypeMap[dataType] for dataType in colDataType]
    # acquire the column names in Oracle DB
    oracleColNames = [x[0].lower() for x in oracleCursor.description]

    # create table  if it does not exist
    casSession.execute('''CREATE TABLE IF NOT EXISTS {}.{} ({}, PRIMARY KEY (rowid))'''.format(\
        row['cassandraKeyspace'].lower(), row['cassandraTblName'].lower(),\
            ','.join([x + ' ' + y for (x,y) in zip(oracleColNames, cassType)])))

    #%% upload data to cassandra
    insertCQL = """INSERT INTO {}.{} ({}) VALUES ({})""".format(\
                row['cassandraKeyspace'].lower(), row['cassandraTblName'].lower(), \
                ','.join(oracleColNames), \
                ','.join(['%s' for x in range(len(oracleColNames))]))
    preparedCQL = casSession.prepare(insertCQL.replace('%s', '?'))

    numRowsOracle = len(oracleRows)
    while len(oracleRows) is not 0:
        oracleRows = oracleCursor.fetchmany(numFetch)
        cassandra.concurrent.execute_concurrent_with_args(casSession, preparedCQL, oracleRows, concurrency = 50)
        numRowsOracle += len(oracleRows)                   

    # shutdown the connection to cassandra
    casCon.shutdown()
    # clost the connection to Oracle
    oracleCursor.close()
    oracleConn.close()


def main(argv):
    
    #%% input check
    if len(argv) is 0:
        tableListFN = 'dataTable.csv'
        print """The default filename of the table list is '%s'.""" % tableListFN
    elif len(argv) > 1:
        raise TooManyArgsError('Too many input arguments!')
    else:
        tableListFN = argv[0]

    #%% read input file
    # print log
    print 'The filename you input is %s.' % tableListFN
    # read the title row of csv
    fieldnames = None
    with open(tableListFN) as csvfile:
        firstRow = csvfile.readlines(1)
        fieldnames = tuple(firstRow[0].strip('\n').split(","))

    # read the rows of csv
    tableList = list()
    with open(tableListFN) as csvfile:
        for row in islice(csvfile, 1, None):
            values = [elem.upper() for elem in row.strip('\n').split(",")]
            tableList.append(dict(zip(fieldnames, values)))

    if len(tableList) is 0:
        print 'There is no data in %s.' % tableListFN

    jobs = []
    job_server = pp.Server()
    job_server.set_ncpus()
    for row in tableList:
        jobs.append(job_server.submit(implementUploadTbl, (row,)))
    runJob = [job() for job in jobs]
    
if __name__ == "__main__":
    import time    
    st = time.clock()
    main(sys.argv[1:])
    print 'The total upload time is %s seconds...\n' % (time.clock() - st)
```