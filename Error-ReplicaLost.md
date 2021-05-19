# All replicas are lost
Recently I restored many parts using clickhouse-backup, it attaches partitions to ClickHouse. The parts has already attached successfully, but some of the replicas doesn't auto sync up the data. When I check the zookeeper log(your_part_znode/log) and queue(your_part_znode/replicas/machine/queue), there are many logs queued. When I check clickhouse log, it shows following error:

```
2021.05.14 04:15:59.949972 [ 87343 ] {} <Error> testdb.testtable (ReplicatedMergeTreeRestartingThread): void DB::ReplicatedMergeTreeRestartingThread::run(): Code: 415, e.displayText() = DB::Exception: All replicas are lost, Stack trace (when copying this message, always include the lines below):

0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0x87f714a in /usr/bin/clickhouse
1. DB::StorageReplicatedMergeTree::cloneReplicaIfNeeded(std::__1::shared_ptr<zkutil::ZooKeeper>) @ 0xfa16d50 in /usr/bin/clickhouse
2. DB::ReplicatedMergeTreeRestartingThread::tryStartup() @ 0xfdcdc0a in /usr/bin/clickhouse
3. DB::ReplicatedMergeTreeRestartingThread::run() @ 0xfdcced8 in /usr/bin/clickhouse
4. DB::BackgroundSchedulePoolTaskInfo::execute() @ 0xf0cbf80 in /usr/bin/clickhouse
5. DB::BackgroundSchedulePool::threadFunction() @ 0xf0cdf77 in /usr/bin/clickhouse
6. ? @ 0xf0ced42 in /usr/bin/clickhouse
7. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0x88372bf in /usr/bin/clickhouse
8. ? @ 0x883ade3 in /usr/bin/clickhouse
9. start_thread @ 0x76db in /lib/x86_64-linux-gnu/libpthread-2.27.so
10. __clone @ 0x12188f in /lib/x86_64-linux-gnu/libc-2.27.so
 (version 21.4.6.55 (official build))
```

# Quick "fix"
The printed error stack trace is very important and useful information for us to debug.

The error is thrown at StorageReplicatedMergeTree::cloneReplicaIfNeeded, we can check the [source code](https://clickhouse.tech/codebrowser/html_report/ClickHouse/src/Storages/StorageReplicatedMergeTree.cpp.html#_ZN2DB26StorageReplicatedMergeTree13dropPartitionERKNSt3__110shared_ptrINS_4IASTEEEbbNS2_INS_7ContextEEEb). From the source code, we found the "is_lost" znode will mark whether the replica is lost, but for the machines I have attatched data, I don't think it is lost, and the data should be transfered to its replicas.

## Modify znode is_lost to 0
I checked the is_lost znode(*zookeepr_part_znode*/replicas/*your_machine*/is_lost), all the replicas' is_lost is marked as 1. So I tried to mark the is_lost to 0 for the machines have data, then the data is automatically synced up. *(If you don't know how to get/set the znode information, the below section gives you the link to the zookeeper documents)*

## For new zookeeper users
Zookeeper provides client to check the znode informations, if you are new users to zookeeper, please learn from their documents [here](https://zookeeper.apache.org/doc/r3.3.3/zookeeperStarted.html).

# Why this error happened?
I guess it may caused by the too many queued messagegs, when it meets the maximum number, the new message cannot be inserted. So it marks the machine as lost. From comments at this [link](https://github.com/ClickHouse/ClickHouse/issues/4112)

```
Replica is marked as lost (in zookeeper, there is is_lost flag for each replica) if it isn't active and it's replication queue has more than max_replicated_logs_to_keep (= 10000) items. If, for example, you delete information from zookeeper for all replicas which are not lost, you'll get this message.
```
