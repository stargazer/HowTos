## MySQL bottlenecks

### Storage Engine

Do you use the ``InnoDB`` storage engine? If not, is there a good reason for doing so? Other than the well-known reasons for using ``InnoDB``(real foreign key constraints, data integrity, ACID compliance, transaction support etc), one can get much more detailed insight from the slow query log and the utilities of the [Percona Toolkit](http://www.percona.com/software/percona-toolkit)

### Identify slow queries

Activate MySQL's [Slow Query Log](https://dev.mysql.com/doc/refman/5.0/en/slow-query-log.html). Analyze the findings using Percona's [``pt-query-digest``](http://www.percona.com/doc/percona-toolkit/2.2/pt-query-digest.html)

### Understand the query's execution plan
Once you have identified a slow query, use MySQL's ``EXPLAIN`` statement to analyze its query execution plan

    EXPLAIN <query>

* Maybe too many rows are scanned?
* Temporary tables created?
* Poor use of indexes for selecting or ordering?
* Bad ordering of joins by the MySQL optimizer?    

### Profiling queries

``EXPLAIN`` simply shows how the MySQL optimizer *plans* to execute the query. In order to see how the query was actually executed, we need to perform *profiling*.

Connect to the MySQL server

    mysql -u<user> -p<password> <database>

Enable profiling for this session

    SET profiling = 1;

Run the query you'd like to profile

    ...

Show all query profiles and find the ``Query_ID`` of the query you'd like to profile.

    SHOW PROFILES;

Assign that to the ``query_id`` parameter

    SET @query_id = <Query_ID>;

Disable profiling

    SET profiling=0;

RUN the following query, for a detailed breakdown of the various steps of the query with ID ``@query_id``    
    
    SELECT STATE, SUM(DURATION) AS Total_R, ROUND(100 * SUM(DURATION) / (SELECT SUM(DURATION) FROM  INFORMATION_SCHEMA.PROFILING WHERE QUERY_ID= @query_id) ,2) AS Pct_R, COUNT(*) AS Calls, SUM(Duration) / COUNT(*) AS "R/Call"
    FROM INFORMATION_SCHEMA.PROFILING
	WHERE QUERY_ID = @query_id
	GROUP BY STATE
	ORDER BY Total_R DESC;            

Where does the query spend most of its time?
 
* Joining?
* Temporary tables?    
* Sending items? 
* Sorting results?    

### Study the whole system's behavior

* Set triggers using [``pt-stalk``](http://www.percona.com/doc/percona-toolkit/2.2/pt-stalk.html) and analyze the system's state on each identified peak, by using [``pt-sift``](http://www.percona.com/doc/percona-toolkit/2.2/pt-sift.html).

### What can we do?

* Scale up the server to buy us some time? Extra memory or CPU power?
* Better indexing?
* Better use of joins?    
* Rewrite queries?
* Cache fields of create summary tables?
* Defer some fields from database queries? Especially those that are expensive to retrieve and send.
* Study Query cache ratios. Is it worth having it? Can we configure it better? Maybe disable it altogether?

