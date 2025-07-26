📖 [中文文档](./README_cn.md)

This tool is used to online retrieve the size, start time, end time, and duration of MySQL binlogs.

### In what scenarios might you use this tool?

- **Cloud service scenario**: When you cannot log in to the MySQL server to view the binlog timestamp information.
- **Replication delay**: This tool can be used to check the size of the binlog or the write volume of binlogs during a certain time period when there is replication delay.
- **Point-in-time recovery**: This tool can help identify the corresponding binlog files based on the approximate time of operations.

Let’s take a look at the tool in action.

```bash
./mysql-binlog-time-extractor -h 10.0.0.108 -P 3306 -u root -p 123456
+------------------+--------------------+---------------------+---------------------+-----------+---------+
|     Log_name     |     File_size      |     Start_time      |      End_time       | Duration  |  GTID   |
+------------------+--------------------+---------------------+---------------------+-----------+---------+
| mysql-bin.000046 | 805 (805.00 bytes) | 2025-06-22 11:09:38 | 2025-06-24 10:33:59 | 47:24:21  | 503-504 |
| mysql-bin.000047 | 12103 (11.82 KB)   | 2025-06-24 10:33:59 | 2025-07-05 00:02:27 | 253:28:28 | 505-517 |
| mysql-bin.000048 | 261 (261.00 bytes) | 2025-07-05 00:02:27 | 2025-07-10 15:03:01 | 135:00:34 |         |
| mysql-bin.000049 | 261 (261.00 bytes) | 2025-07-10 15:03:01 | 2025-07-10 15:05:29 | 00:02:28  |         |
| mysql-bin.000050 | 9074 (8.86 KB)     | 2025-07-10 15:05:29 | 2025-07-23 12:20:32 | 309:15:03 | 518-550 |
| mysql-bin.000051 | 586710 (572.96 KB) | 2025-07-23 12:20:32 | 2025-07-24 08:48:08 | 20:27:36  | 551-754 |
| mysql-bin.000052 | 464 (464.00 bytes) | 2025-07-24 08:48:08 |                     |           |         |
+------------------+--------------------+---------------------+---------------------+-----------+---------+
```

### Tool URL

Project URL: https://github.com/slowtech/mysql-binlog-time-extractor

You can use the binary package directly or compile from source.

#### Direct Use of Binary Package

```bash
# wget https://github.com/slowtech/mysql-binlog-time-extractor/releases/download/v1.0.0/mysql-binlog-time-extractor-linux-amd64.tar.gz
# tar xvf mysql-binlog-time-extractor-linux-amd64.tar.gz
```

After extraction, a file named `mysql-binlog-time-extractor` will be generated in the current directory.

#### Compile from Source

```bash
# wget https://github.com/slowtech/mysql-binlog-time-extractor/archive/refs/tags/v1.0.0.tar.gz
# tar xvf v1.0.0.tar.gz
# cd mysql-binlog-time-extractor-1.0.0/
# go build
```

After compiling, a file named `mysql-binlog-time-extractor` will be generated in the current directory.

### Parameter Explanation

```bash
# ./mysql-binlog-time-extractor --help
Usage of ./mysql-binlog-time-extractor:
  -P int
        MySQL port (default 3306)
  -h string
        MySQL host (default "localhost")
  -n int
        Number of goroutines to run concurrently (default 5)
  -p string
        MySQL password
  -u string
        MySQL user (default "root")
  -v    Enable verbose logging
```

- **`-h`, `-P`, `-u`, `-p`**: Used to specify the instance's IP, port, username, and password. If `-p` is not specified, the tool will prompt for the password.
- **`-n`**: The number of concurrent goroutines (default is 5), which means 5 binlogs will be analyzed concurrently. If there are a large number of binlogs, you can increase the number of goroutines to improve efficiency.
- **`-v`**: Enable verbose logging, which will show the analysis progress. For example:

```bash
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go SHOW BINARY LOGS done, 7 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000052 done, still 6 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000051 done, still 5 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000050 done, still 4 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000049 done, still 3 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000048 done, still 2 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000047 done, still 1 binlogs to analyze
[2025/07/25 00:02:29] [info] mysql_binlog_time_extractor.go mysql-bin.000046 done, still 0 binlogs to analyze
```

### Implementation Details

1. First, execute `SHOW BINARY LOGS` to retrieve the list of all binlog files, including their filenames and sizes.
2. The tool masquerades as a replica and analyzes the first two events of each binlog file: `FORMAT_DESCRIPTION_EVENT` and `PREVIOUS_GTIDS_EVENT`.
   - The `FORMAT_DESCRIPTION_EVENT` records the creation time of the binlog.
   - The `PREVIOUS_GTIDS_EVENT` records the set of GTIDs before the current binlog.
3. To improve analysis efficiency and reduce the impact on the master server, the tool only analyzes the first two events of each binlog. How is the end time of the current binlog determined? A clever approach is used: the creation time of the next binlog is used as the end time for the current binlog, rather than scanning all events to get the time of the last event.

### Notes

During execution, the tool may print the following error, indicating that the binlog streamer has already been closed. This error originates from the third-party library `github.com/go-mysql-org/go-mysql/replication` and does not affect the tool's normal operation. You can safely ignore it.

```bash
[2025/07/25 02:57:38] [error] binlogstreamer.go:78 close sync with err: sync is been closing...
```
