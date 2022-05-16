# Query Analyzer Agent - Capture and analyze the queries without overhead.

Query Analyzer Agent runs on the database server. It captures all the queries by sniffing the network port, aggregates the queries and sends results to a remote server for further analysis. Refer to [LinkedIn's Engineering Blog](https://engineering.linkedin.com/blog/2017/09/query-analyzer--a-tool-for-analyzing-mysql-queries-without-overh) for more details.

## Getting Started

### Prerequisites

Query Analyzer Agent is written in Go, so before you get started you should [install and setup Go](https://golang.org/doc/install). You can also follow the steps here to install and setup Go.

```
$ wget https://dl.google.com/go/go1.18.linux-amd64.tar.gz
$ sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
$ echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
$ go version
```

Query Analyzer Agent requires the following external libraries

- pcap.h (provided by libpcap-dev package), gcc or build-essential for building this package
  - RHEL/CentOs/Fedora:
    ```
    $ sudo yum install gcc libpcap libpcap-devel git
    ```
  - Debian/Ubuntu:
    ```
    $ sudo apt-get install build-essential libpcap-dev git
    ```

Docker to run mysql server

```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sh get-docker.sh
$ docker run --name mysql3307 -p3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
$ docker run --name mysql3308 -p3308:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
$ docker run --name adminer -p8088:8080 -d adminer
$ import `sql/remote_schema.sql` to mysql3308
```

visit http://localhost:8088/ , use `LanIP:3307` (eg. 192.168.0.2:3307) as database host

## Build

```
$ git clone https://github.com/linkedin/QueryAnalyzerAgent
$ cd QueryAnalyzerAgent
$ go build -o QueryAnalyzerAgent
$ ./QueryAnalyzerAgent
```

## Query Analytics

Once you understand the schema, you can write queries and build fancy UI to extract the information you want.
Examples:

- Top 5 queries which have the maximum total run time during a specific interval. If a query takes 1 second and executes 1000 times, the total run time is 1000 seconds.

  ```
  SELECT
      SUM(count),
      SUM(querytime)
  INTO
      @count, @qt
  FROM
      query_history history
  WHERE
      history.hostname='mysql.database-server-001.linkedin.com'
      AND ts>='2020-03-11 09:00:00'
      AND ts<='2020-03-11 10:00:00';

  SELECT
      info.checksum,
      info.firstseen AS first_seen,
      info.fingerprint,
      info.sample,
      SUM(count) as count,
      ROUND(((SUM(count)/@count)*100),2) AS pct_total_query_count,
      ROUND((SUM(count)/(TIME_TO_SEC(TIMEDIFF(MAX(history.ts),MIN(history.ts))))),2) AS qps,
      ROUND((SUM(querytime)/SUM(count)),6) AS avg_query_time,
      ROUND(SUM(querytime),6) AS sum_query_time,
      ROUND((SUM(querytime)/@qt)*100,2) AS pct_total_query_time,
      MIN(info.mintime) AS min_query_time,
      MAX(info.maxtime) AS max_query_time
  FROM
      query_history history
  JOIN
      query_info info
  ON
      info.checksum=history.checksum
      AND info.hostname=history.hostname
  WHERE
      info.hostname='mysql.database-server-001.linkedin.com'
      AND ts>='2020-03-11 09:00:00'
      AND ts<='2020-03-11 10:00:00'
  GROUP BY
      info.checksum
  ORDER BY
      pct_total_query_time DESC
  LIMIT 5\G
  ```

- Trend for a particular query
  ```
  SELECT
      UNIX_TIMESTAMP(ts),
      ROUND(querytime/count,6)
  FROM
      query_history history
  WHERE
      history.checksum='D22AB75FA3CC05DC'
      AND history.hostname='mysql.database-server-001.linkedin.com'
      AND ts>='2020-03-11 09:00:00'
      AND ts<='2020-03-11 10:00:00';
  ```
- Queries fired from a particular IP
  ```
  SELECT
      info.checksum,
      info.fingerprint,
      info.sample
  FROM
      query_history history
  JOIN
      query_info info
  ON
      info.checksum=history.checksum
      AND info.hostname=history.hostname
  WHERE
      history.src='10.251.225.27'
  LIMIT 5;
  ```
- New queries on a particular day
  ```
  SELECT
      info.firstseen,
      info.checksum,
      info.fingerprint,
      info.sample
  FROM
      query_info info
  WHERE
      info.hostname = 'mysql.database-server-001.linkedin.com'
      AND info.firstseen >= '2020-03-10 00:00:00'
      AND info.firstseen < '2020-03-11 00:00:00'
  LIMIT 5;
  ```

## Limitations

- As of now, it works only for MySQL.

- Does not account for

  - SSL
  - Compressed packets
  - Replication traffic
  - Big queries for performance reasons

- The number of unique query fingerprints should be limited (like <100K). For example if there is some blob in the query and the tool is unable to generate the correct fingerprint, it will lead to a huge number of fingerprints and can increase the memory footprint of QueryAnalyzerAgent.<br /><br />
  Another example is if you are using Github's Orchestrator in pseudo GTID mode, it generates queries like

  ```
  drop view if exists `_pseudo_gtid_`.`_asc:5d8a58c6:0911a85c:865c051f49639e79`
  ```

  The fingerprint for those queries will be unique each time and it will lead to more number of distinct queries in QueryAnalyzerAgent. Code to ignore those queries is commented, uncomment if needed.

- Test the performance of QueryAnalyzerAgent in your staging environment before running on production.
