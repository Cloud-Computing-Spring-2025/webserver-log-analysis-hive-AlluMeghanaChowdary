# 📄 Web Server Log Analysis Using Apache Hive

## 📌 Project Overview

This project analyzes web server logs using Apache Hive. It processes structured log data from CSV format, performs SQL-based analytics, and extracts meaningful insights such as page visits, HTTP status codes, user agents, traffic trends, and suspicious activities. The project also optimizes performance using Hive partitioning.

## 🛠 Implementation Approach

The project leverages Apache Hive to process and analyze web server logs stored in HDFS. The dataset, structured in CSV format, is first loaded into Hive after creating an external table. A series of HiveQL queries are then executed to extract meaningful insights. These queries include counting total web requests, analyzing HTTP status codes, identifying the most visited pages, and detecting suspicious activities based on failed request patterns. Additionally, partitioning by status code is implemented to optimize query performance by reducing scan time and improving retrieval efficiency. The results are then exported from Hive, stored in HDFS, and later copied to the local system using Docker for further analysis.

## ⚙️ Execution Steps

#### 1️⃣ Start Hadoop & Hive Services

```bash
start-dfs.sh
start-yarn.sh
hive
```

#### 2️⃣ Create Hive Database & Table

```sql
CREATE DATABASE web_logs_db;
USE web_logs_db;
```
```sql
CREATE EXTERNAL TABLE web_logs (
    ip STRING,
    timestamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/web_logs';
```

#### 3️⃣ Load Data into HDFS & Hive

```bash
hdfs dfs -mkdir -p /user/hive/web_logs
hdfs dfs -put web_logs.csv /user/hive/web_logs
```

```sql
LOAD DATA INPATH '/user/hive/web_logs/web_logs.csv' INTO TABLE web_logs;
```

#### 4️⃣ Execute Queries
✅ Total Web Requests
```sql
SELECT COUNT(*) AS total_requests FROM web_logs;
```
✅ Status Code Analysis
```sql
SELECT status, COUNT(*) AS count FROM web_logs GROUP BY status;
```
✅ Most Visited Pages
```sql
SELECT url, COUNT(*) AS visit_count 
FROM web_logs 
GROUP BY url 
ORDER BY visit_count DESC 
LIMIT 3;
```
✅ Traffic Source Analysis
```sql
SELECT user_agent, COUNT(*) AS count 
FROM web_logs 
GROUP BY user_agent 
ORDER BY count DESC;
```
✅ Detect Suspicious IPs
```sql
SELECT ip, COUNT(*) AS failed_requests 
FROM web_logs 
WHERE status IN (404, 500) 
GROUP BY ip 
HAVING failed_requests > 3;
```
✅ Traffic Trends
```sql
SELECT SUBSTR(timestamp, 1, 16) AS time_minute, COUNT(*) AS request_count 
FROM web_logs 
GROUP BY time_minute 
ORDER BY time_minute;
```
#### 5️⃣ Implement Partitioning
```sql
CREATE TABLE web_logs_partitioned (
    ip STRING,
    `timestamp` STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```
```sql
INSERT INTO TABLE web_logs_partitioned PARTITION(status)
SELECT ip, timestamp, url, user_agent, status FROM web_logs;
```

#### 6️⃣ Export Hive Output from HDFS to Local Machine
Copy from HDFS to Local Inside Docker Container
```bash
hdfs dfs -get /output /opt/hive-output/
```
Copy from Docker Container to Host Machine
```bash
exit
docker cp hive-server:/opt/hive-output/ shared-folder/output/
```

## 🎯 Challenges Faced

Several challenges were encountered while implementing the project. One of the main issues was data formatting; the CSV file had inconsistencies, requiring proper field delimiters to ensure correct parsing in Hive. Additionally, query performance was initially slow due to the large dataset, which was mitigated by using partitioning techniques. Another significant challenge was file handling in HDFS, as files were sometimes misplaced or not recognized by Hive, necessitating verification with hdfs dfs -ls commands before loading. Exporting Hive query results from HDFS to a local system also posed a challenge due to permission restrictions, which was resolved by adjusting Docker container access and ensuring proper use of hdfs dfs -get. By overcoming these issues, the project successfully optimized the analysis of web server logs, making it efficient and scalable.

## 📊 Sample Input and Expected Output
✅ Sample Input (web_logs.csv)
```bash
ip,timestamp,url,status,user_agent
192.168.1.5,2024-02-02 08:30:00,/dashboard,200,Firefox/95.0
192.168.1.8,2024-02-02 08:32:00,/login,200,Edge/110.0
192.168.1.12,2024-02-02 08:35:00,/profile,500,Chrome/91.0
192.168.1.22,2024-02-02 08:40:00,/home,404,Safari/14.0
192.168.1.5,2024-02-02 08:45:00,/dashboard,200,Firefox/95.0
192.168.1.8,2024-02-02 08:50:00,/home,404,Edge/110.0
192.168.1.12,2024-02-02 08:55:00,/checkout,500,Chrome/91.0
```
✅ Expected Output

Total Web Requests

#### Total_requests
```bash
100
```
#### Status Code Analysis
```bash
status | count
-----------------
200    | 60
404    | 20
500    | 20
```

#### Most Visited Pages
```bash
url          | visit_count
--------------------------
/dashboard   | 25
/login       | 20
/profile     | 15
```

#### Traffic Source Analysis
```bash
user_agent    | count
-----------------------
Firefox/95.0  | 40
Chrome/91.0   | 35
Edge/110.0    | 25
```
#### Suspicious IP Addresses
```bash 
ip             | failed_requests
--------------------------------
192.168.1.12   | 4
192.168.1.22   | 5
```

#### Traffic Trends Over Time
```bash
time_minute         | request_count
---------------------------------
2024-02-02 08:30    | 3
2024-02-02 08:35    | 5
2024-02-02 08:40    | 6
```