# mysql-binlog-time-extractor

去年写的一个小工具，用于在线获取 MySQL binlog 的大小、开始时间、结束时间和持续时长。

什么场景下会用上这个工具呢？

1. 云服务场景，无法登录 MySQL 服务器查看 binlog 的时间戳信息。
2. 主从延迟时，可以使用这个工具来查看 binlog 的大小或者某个时间段 binlog 的写入量。
3. 基于时间点的恢复时，可以根据操作的大致时间来定位对应的 binlog 文件。

不多说，直接看看工具的效果。

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



# 工具地址

项目地址：https://github.com/slowtech/mysql-binlog-time-extractor

可直接使用二进制包，也可以源码编译。

### 直接使用二进制包

```bash
# wget https://github.com/slowtech/mysql-binlog-time-extractor/releases/download/v1.0.0/mysql-binlog-time-extractor-linux-amd64.tar.gz
# tar xvf mysql-binlog-time-extractor-linux-amd64.tar.gz
```

解压后，会在当前目录生成一个名为`mysql-binlog-time-extractor`的可执行文件。

### 源码编译

```bash
# wget https://github.com/slowtech/mysql-binlog-time-extractor/archive/refs/tags/v1.0.0.tar.gz
# tar xvf v1.0.0.tar.gz 
# cd mysql-binlog-time-extractor-1.0.0/
# go build
```

编译完成后，会在当前目录生成一个名为`mysql-binlog-time-extractor`的可执行文件。



# 参数解析

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

其中，-h、-P、-u、-p 分别用来指定实例的 IP、端口、用户名和密码。如果不指定 -p，则会提示输入密码。

-n 是并发数，默认是 5，即每次会同时分析 5 个 binlog。在 binlog 数量较多的情况下，可以适当增加并发数来提高分析效率。

-v 打印分析进度，例如，

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



# 实现原理

1. 首先执行 `SHOW BINARY LOGS`，获取所有 binlog 文件的列表，包括文件名和文件大小。

2. 将自己“伪装”为从库，逐个分析每个 binlog 文件的前两个事件：`FORMAT_DESCRIPTION_EVENT` 和 `PREVIOUS_GTIDS_EVENT`。

   其中，`FORMAT_DESCRIPTION_EVENT` 记录了 binlog 的创建时间，`PREVIOUS_GTIDS_EVENT` 记录了当前 binlog 之前的所有 GTID 集合。

3. 为了提升分析效率并减少对主库的影响，该工具只会分析每个 binlog 文件的前两个事件。那么，如何确定当前 binlog 的结束时间呢？这里采用了一种巧妙的方式：直接使用下一个 binlog 的创建时间作为当前 binlog 的结束时间，而不是扫描所有事件来获取最后一个事件的时间。

   

# 注意事项

工具执行过程中，可能会打印以下错误，表示在尝试关闭 binlog streamer 时，发现其已处于关闭状态。该错误来自第三方库 `github.com/go-mysql-org/go-mysql/replication`，不影响工具的正常使用，可忽略。

```bash
[2025/07/25 02:57:38] [error] binlogstreamer.go:78 close sync with err: sync is been closing...
```
