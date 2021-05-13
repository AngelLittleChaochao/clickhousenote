# Deadlock error
Recently we encountered the deadlock error very often, the error message looks like below. And the error also exists even I tried many times again. What's the problems? How to fix it workaround?
```
2021.05.10 09:01:11.922466 [ 76138 ] {43ecd777-824c-47f4-b5d3-41d74158353d} <Error> TCPHandler: Code: 473, e.displayText() = DB::Exception: WRITE locking attempt on "myprod.mytest" has timed out! (120000ms) Possible deadlock avoided. Client should retry., Stack trace:

0. DB::IStorage::tryLockTimed(std::__1::shared_ptr<DB::RWLockImpl> const&, DB::RWLockImpl::Type, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::chrono::duration<long long, std::__1::ratio<1l, 1000l> > const&) const @ 0xf031f65 in /usr/bin/clickhouse
1. DB::IStorage::lockForAlter(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::chrono::duration<long long, std::__1::ratio<1l, 1000l> > const&) @ 0xf032522 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::execute() @ 0xeb3fb07 in /usr/bin/clickhouse
3. ? @ 0xeeb94f2 in /usr/bin/clickhouse
4. DB::executeQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::Context&, bool, DB::QueryProcessingStage::Enum, bool) @ 0xeeb7e4c in /usr/bin/clickhouse
5. DB::TCPHandler::runImpl() @ 0xf5b2b15 in /usr/bin/clickhouse
6. DB::TCPHandler::run() @ 0xf5c2799 in /usr/bin/clickhouse
7. Poco::Net::TCPServerConnection::start() @ 0x11b6018f in /usr/bin/clickhouse
8. Poco::Net::TCPServerDispatcher::run() @ 0x11b61ba1 in /usr/bin/clickhouse
9. Poco::PooledThread::run() @ 0x11c98c49 in /usr/bin/clickhouse
10. Poco::ThreadImpl::runnableEntry(void*) @ 0x11c94aaa in /usr/bin/clickhouse
11. start_thread @ 0x76db in /lib/x86_64-linux-gnu/libpthread-2.27.so
12. __clone @ 0x12188f in /lib/x86_64-linux-gnu/libc-2.27.so

```

# Quick "fix"
I don't think this method can be called fix, since it doesnot fix the problem, it is just to <strong> <em>restart the clickhouse </em> </strong>. After restarting clickhouse, the error will disapper.

But how to detect the deadlock problem(since it may need some days to fix the problem) and restart automatically? So other task will not be blocked like inserting new data.

Clickhouse provides many "log" tables, it makes you to learn clickhouse state in many dimensions. If you want to learn some metrics of clickhouse, try to find whether that has table storing the information.

For the deadlock error, you can find in the [system.errors](!https://clickhouse.tech/docs/en/operations/system-tables/errors/) table. System.error table contains error codes with the number of times they have been triggered. In my system.error table, there are columns like:
* name (String) — name of the error (errorCodeToName).
* code (Int32) — code number of the error.
* value (UInt64) — the number of times this error has been happened.

For the deadlock avoided error, the name is DEADLOCK_AVOIDED, if this error exits in the system.errors and the value is greater than 0, you can try to restart clickhouse.

My system.errors tables has the following information:
```
SELECT
    name,
    code,
    value
FROM system.errors

Query id: 6386d49b-ae06-4863-9bc4-e30c91abc59c

┌─name────────────────┬─code─┬─value─┐
│ UNKNOWN_IDENTIFIER  │   47 │ 10297 │
│ NO_REPLICA_HAS_PART │  234 │  1208 │
│ DEADLOCK_AVOIDED    │  473 │     8 │
└─────────────────────┴──────┴───────┘
```
All the clickhouse error code, please find [here](!https://github.com/ClickHouse/ClickHouse/blob/0264124146f1d3377c448eac66975a112971417a/src/Common/ErrorCodes.cpp).

You can also find which query blocks and other query cannot get the lock? Query the [processes](!https://clickhouse.tech/docs/en/operations/system-tables/processes/) table(you can also use show processlist command). Using query like to see how long time the query has executed:

```
SELECT elapsed, query from system.processes
```

# Read the code
When you see errors, clickhouse also give the call stack. It's very helpful for developers to debug and find the root cause. For this deadlock problem, it's easy to find the call stack to see which line prints the error. But this error is not the root cause. The root cause is other query has got the lock and keept the lock for very long time. In my case for the ReplicatedMergedTree table, it is caused by the 'DROP PARTITION' sql. The zookeeper may has lost some message, and clickhouse server thread is waiting for task in "Queue" but the task has lost.

[StorageReplicatedMergeTree](https://clickhouse.tech/codebrowser/html_report/ClickHouse/src/Storages/StorageReplicatedMergeTree.cpp.html#_ZN2DB26StorageReplicatedMergeTree13dropPartitionERKNSt3__110shared_ptrINS_4IASTEEEbbNS2_INS_7ContextEEEb)

Clickhouse provided the online compiled code, it's very convinient for us to explore code.



