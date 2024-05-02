# trino2trino

## Basic usage/testing

Build the plugin with
```shell
mvn clean install -DskipTests
```
There's some issue with the test suite that means they're not running properly at the moment, so we need to skip them.

When the plugin is built, start the docker compose file:
```shell
docker compose up -d
```

You can now shell into the trino-coordinator container to run commands:
```shell
docker exec -it trino-coordinator trino
```

```
trino> SHOW catalogs;
 Catalog 
---------
 system  
 tpch    
 trino   
(3 rows)

Query 20240502_090011_00000_vbcrh, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
6.20 [0 rows, 0B] [0 rows/s, 0B/s]

trino> SHOW schemas in trino;
       Schema       
--------------------
 information_schema 
 sf1                
 sf10               
 sf100              
 sf1000             
 sf10000            
 sf100000           
 sf300              
 sf3000             
 sf30000            
 tiny               
(11 rows)

Query 20240502_090026_00001_vbcrh, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
6.65 [11 rows, 128B] [1 rows/s, 19B/s]

trino> SELECT * FROM trino.tiny.customer LIMIT 2;
 c_customer_sk |  c_customer_id   | c_current_cdemo_sk | c_current_hdemo_sk | c_current_addr_sk | c_first_shipto_date_sk | c_first_sales_date_sk | c_salutation |    >
---------------+------------------+--------------------+--------------------+-------------------+------------------------+-----------------------+--------------+---->
             1 | AAAAAAAABAAAAAAA |             980124 |               7135 |               946 |                2452238 |               2452208 | Mr.          | Jav>
             2 | AAAAAAAACAAAAAAA |             819667 |               1461 |               655 |                2452318 |               2452288 | Dr.          | Amy>
(2 rows)

Query 20240502_090154_00004_vbcrh, FINISHED, 1 node
Splits: 18 total, 18 done (100.00%)
2.99 [1000 rows, 0B] [334 rows/s, 0B/s]
```

and if you check the logs of the remote trino, you'll see its run those queries:
```shell
docker logs --tail 10 trino-coordinator-remoter
```

```shell
2024-05-02T08:58:26.993Z        INFO    main    io.trino.server.Server  ======== SERVER STARTED ========
2024-05-02T09:00:32.479Z        INFO    dispatcher-query-2      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090027_00000_3g7h6 :: FINISHED :: elapsed 5021ms :: planning 1517ms :: waiting 85ms :: scheduling 2590ms :: running 108ms :: finishing 806ms :: begin 2024-05-02T09:00:27.345Z :: end 2024-05-02T09:00:32.366Z
2024-05-02T09:01:25.918Z        INFO    dispatcher-query-5      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090124_00001_3g7h6 :: FINISHED :: elapsed 1298ms :: planning 370ms :: waiting 32ms :: scheduling 550ms :: running 50ms :: finishing 328ms :: begin 2024-05-02T09:01:24.581Z :: end 2024-05-02T09:01:25.879Z
2024-05-02T09:01:26.668Z        INFO    dispatcher-query-6      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090125_00002_3g7h6 :: FINISHED :: elapsed 671ms :: planning 68ms :: waiting 5ms :: scheduling 106ms :: running 207ms :: finishing 290ms :: begin 2024-05-02T09:01:25.952Z :: end 2024-05-02T09:01:26.623Z
2024-05-02T09:01:41.308Z        INFO    dispatcher-query-3      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090140_00003_3g7h6 :: FINISHED :: elapsed 409ms :: planning 52ms :: waiting 13ms :: scheduling 91ms :: running 106ms :: finishing 160ms :: begin 2024-05-02T09:01:40.879Z :: end 2024-05-02T09:01:41.288Z
2024-05-02T09:01:42.560Z        INFO    dispatcher-query-3      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090141_00004_3g7h6 :: FINISHED :: elapsed 607ms :: planning 96ms :: waiting 8ms :: scheduling 263ms :: running 67ms :: finishing 181ms :: begin 2024-05-02T09:01:41.934Z :: end 2024-05-02T09:01:42.541Z
2024-05-02T09:01:55.486Z        INFO    dispatcher-query-7      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090154_00005_3g7h6 :: FINISHED :: elapsed 471ms :: planning 79ms :: waiting 15ms :: scheduling 194ms :: running 45ms :: finishing 153ms :: begin 2024-05-02T09:01:54.986Z :: end 2024-05-02T09:01:55.457Z
2024-05-02T09:01:56.136Z        INFO    dispatcher-query-2      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090155_00006_3g7h6 :: FINISHED :: elapsed 599ms :: planning 102ms :: waiting 14ms :: scheduling 314ms :: running 27ms :: finishing 156ms :: begin 2024-05-02T09:01:55.520Z :: end 2024-05-02T09:01:56.119Z
2024-05-02T09:01:56.480Z        INFO    dispatcher-query-4      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090156_00007_3g7h6 :: FINISHED :: elapsed 17ms :: planning 0ms :: waiting 0ms :: scheduling 17ms :: running 0ms :: finishing 17ms :: begin 2024-05-02T09:01:56.462Z :: end 2024-05-02T09:01:56.479Z
2024-05-02T09:01:57.691Z        INFO    dispatcher-query-1      io.trino.event.QueryMonitor     TIMELINE: Query 20240502_090156_00008_3g7h6 :: FINISHED :: elapsed 1128ms :: planning 32ms :: waiting 8ms :: scheduling 126ms :: running 881ms :: finishing 89ms :: begin 2024-05-02T09:01:56.516Z :: end 2024-05-02T09:01:57.644Z
```

## Query passthrough

This connector also supports query passthrough so that you can force any/all filtering to happen on the remote trino instance, meaning that only the data you need is returned to you. For our case, this was designed to reduce network congestion.

Using query passthrough also lets the user see alternative connectors that are available to them in the remote trino.

```sql
trino> SELECT * FROM TABLE(
    -> trino.system.query(
    -> query => '
    -> SELECT * FROM system.jdbc.catalogs
    -> '
    -> )
    -> );
 table_cat 
-----------
 system    
 tpcds     
(2 rows)
 
trino> SELECT * FROM TABLE(
            -> trino.system.query(
                                   -> query => '
    -> SELECT * FROM system.jdbc.schemas
    -> '
                           -> )
            -> );
table_schem     | table_catalog 
--------------------+---------------
 information_schema | system        
 jdbc               | system        
 metadata           | system        
 runtime            | system        
 information_schema | tpcds         
 sf1                | tpcds         
 sf10               | tpcds         
 sf100              | tpcds         
 sf1000             | tpcds         
 sf10000            | tpcds         
 sf100000           | tpcds         
 sf300              | tpcds         
 sf3000             | tpcds         
 sf30000            | tpcds         
 tiny               | tpcds         
(15 rows)

Query 20240502_090729_00007_vbcrh, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
0.50 [15 rows, 0B] [30 rows/s, 0B/s]
                
trino> SELECT * FROM TABLE(
    -> trino.system.query(
    -> query => '
    -> SELECT * FROM tpcds.tiny.customer LIMIT 4
    -> '
    -> )
    -> );
c_customer_sk |  c_customer_id   | c_current_cdemo_sk | c_current_hdemo_sk | c_current_addr_sk | c_first_shipto_date_sk | c_>
---------------+------------------+--------------------+--------------------+-------------------+------------------------+--->
             1 | AAAAAAAABAAAAAAA |             980124 |               7135 |               946 |                2452238 |   >
             2 | AAAAAAAACAAAAAAA |             819667 |               1461 |               655 |                2452318 |   >
             3 | AAAAAAAADAAAAAAA |            1473522 |               6247 |               572 |                2449130 |   >
             4 | AAAAAAAAEAAAAAAA |            1703214 |               3986 |               558 |                2450030 |   >
(4 rows)

Query 20240502_090843_00010_vbcrh, FINISHED, 1 node
Splits: 1 total, 1 done (100.00%)
0.87 [4 rows, 0B] [4 rows/s, 0B/s]
```

## Join pushdown

This is incomplete and does not feature in this release. We hope to add it at a later date.

## Remote trino access to other connectors without using passthrough

In `/conf/trino/catalog/trino.properties`, change the line
```
connection-url=jdbc:trino://trino-coordinator-remoter:8080/tpcds
```
(which connects specifically to the remote trino TPCDS connector)
to 
```
connection-url=jdbc:trino://trino-coordinator-remoter:8080/
```
You'll need to reload the containers in the compose file so they pick up the new config.

Now, using this new config, exec into the `trino-coordinator` container to run queries.
```sql
trino> show catalogs;
 Catalog 
---------
 system  
 tpch    
 trino   
(3 rows)

Query 20240502_091140_00000_wwqav, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
5.14 [0 rows, 0B] [0 rows/s, 0B/s]

trino> show schemas in trino;
       Schema       
--------------------
 information_schema 
 jdbc               
 metadata           
 runtime            
 sf1                
 sf10               
 sf100              
 sf1000             
 sf10000            
 sf100000           
 sf300              
 sf3000             
 sf30000            
 tiny               
(14 rows)

Query 20240502_091153_00001_wwqav, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
7.73 [14 rows, 162B] [1 rows/s, 21B/s]

trino> SELECT * FROM trino.tiny.customer LIMIT 3;
c_customer_sk |  c_customer_id   | c_current_cdemo_sk | c_current_hdemo_sk | c_current_addr_sk | c_first_shipto_date_sk | c_>
---------------+------------------+--------------------+--------------------+-------------------+------------------------+--->
             1 | AAAAAAAABAAAAAAA |             980124 |               7135 |               946 |                2452238 |   >
             2 | AAAAAAAACAAAAAAA |             819667 |               1461 |               655 |                2452318 |   >
             3 | AAAAAAAADAAAAAAA |            1473522 |               6247 |               572 |                2449130 |   >
(3 rows)

Query 20240502_091604_00002_wwqav, FINISHED, 1 node
Splits: 18 total, 18 done (100.00%)
4.68 [1000 rows, 0B] [213 rows/s, 0B/s]
```

Using this connection pattern obsfucates the connectors on the remote trino side, appearing to the source trino as though all schemas and tables are under a single connector.