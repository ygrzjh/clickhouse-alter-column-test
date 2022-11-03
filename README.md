## Symptom
Exception occurred when execute `ALTER TABLE add/drop column` sql on the replicated database for the use custom zk path ReplicatedMergeTree table(`zk path`: `/clickhouse/tables/{uuid}/{all}`).

e.g:
```sql
clickhouse01 :) ALTER TABLE demo.users_dim_local  ADD COLUMN `remark` String;

┌─shard─┬─replica──────┬─status─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ 01    │ clickhouse01 │ OK     │                   1 │                1 │
└───────┴──────────────┴────────┴─────────────────────┴──────────────────┘
↙ Progress: 1.00 rows, 49.00 B (0.05 rows/s., 2.39 B/s.)  49%
1 row in set. Elapsed: 20.536 sec.

Received exception from server (version 22.10.1):
Code: 49. DB::Exception: Received from localhost:9000. DB::Exception: There was an error on 02|clickhouse02: Cannot execute replicated DDL query, maximum retries exceeded (probably it's a bug): While executing DDLQueryStatus. (LOGICAL_ERROR)
```

## ClickHouse Cluster
- Clickhouse cluster: `2 shards 1 replicas`
- Clickhouse version: `22.10.1.1877`

Clickhouse Cluster xml config
```xml
<clickhouse>
    <remote_servers>
        <!-- 2 shards 1 replicas -->
        <default>
            <shard>
                <replica>
                    <host>clickhouse01</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>clickhouse02</host>
                    <port>9000</port>
                </replica>
            </shard>
        </default>
        <!-- 1 shards all replicas -->
        <all-replicated>
            <shard>
                <replica>
                    <host>clickhouse01</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse02</host>
                    <port>9000</port>
                </replica>
            </shard>
        </all-replicated>
    </remote_servers>
</clickhouse>
```

Cluster Info
```text
SELECT * FROM system.clusters;

┌─cluster────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ all-replicated │         1 │            1 │           1 │ clickhouse01 │ 172.23.0.11  │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ all-replicated │         1 │            1 │           2 │ clickhouse02 │ 172.23.0.12  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ default        │         1 │            1 │           1 │ clickhouse01 │ 172.23.0.11  │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ default        │         2 │            1 │           1 │ clickhouse02 │ 172.23.0.12  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
└────────────────┴───────────┴──────────────┴─────────────┴──────────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘
```

ClickHouse Node macros
- node1
```text
clickhouse01 :) SELECT * FROM system.macros;

┌─macro───┬─substitution─┐
│ all     │ all          │
│ cluster │ default      │
│ index   │ 0101         │
│ replica │ clickhouse01 │
│ shard   │ 01           │
└─────────┴──────────────┘
```
- node2
```text
clickhouse02 :) SELECT * FROM system.macros;

┌─macro───┬─substitution─┐
│ all     │ all          │
│ cluster │ default      │
│ index   │ 0202         │
│ replica │ clickhouse02 │
│ shard   │ 02           │
└─────────┴──────────────┘
```


## Test it
Clickhouse cluster with 2 shards and 1 replicas built with docker-compose.

docker-compose: [clickhouse-alter-column-test](https://github.com/ygrzjh/clickhouse-alter-column-test)

Run single command, and it will copy configs for each node and run clickhouse cluster with docker-compose
```shell
make config up
```

### Init
- Crate Replicated Database demo (2 shards 1 replicas)
```sql
set allow_experimental_database_replicated = 1;
CREATE DATABASE demo on cluster 'default' ENGINE = Replicated('/clickhouse/default/demo', '{shard}', '{replica}');
```

- Create ReplicatedMergeTree Table (1 shard all replicas,use custom zk path)
```sql
CREATE TABLE demo.users_dim_local
(
    `#user_id` String ,
    `#device_id` String ,
    `#created_time` UInt64 ,
    `#updated_time` UInt64,
    `sex` String,
    `user_name` String
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{uuid}/{all}', '{index}', `#updated_time`)
PARTITION BY cityHash64(`#user_id`) % 64
ORDER BY `#user_id`;
```

### Add column case
- Clickhouse01 exec "alter table add column" exception occurred
```sql
clickhouse01 :) ALTER TABLE demo.users_dim_local  ADD COLUMN `remark` String;

┌─shard─┬─replica──────┬─status─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ 01    │ clickhouse01 │ OK     │                   1 │                1 │
└───────┴──────────────┴────────┴─────────────────────┴──────────────────┘
↙ Progress: 1.00 rows, 49.00 B (0.05 rows/s., 2.39 B/s.)  49%
1 row in set. Elapsed: 20.536 sec.

Received exception from server (version 22.10.1):
Code: 49. DB::Exception: Received from localhost:9000. DB::Exception: There was an error on 02|clickhouse02: Cannot execute replicated DDL query, maximum retries exceeded (probably it's a bug): While executing DDLQueryStatus. (LOGICAL_ERROR)
```

Table schema after "Alter table add column" query execution, actually, clickhouse 2 nodes successfully added `remark` column.
- ClickHouse node1 table schema
```sql
clickhouse01 :) show create table demo.users_dim_local;

SHOW CREATE TABLE demo.users_dim_local

Query id: 5c1ad009-944f-44d9-9907-b95039761771

┌─statement──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ CREATE TABLE demo.users_dim_local
(
    `#user_id` String,
    `#device_id` String,
    `#created_time` UInt64,
    `#updated_time` UInt64,
    `sex` String,
    `user_name` String,
    `remark` String
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{uuid}/{all}', '{index}', `#updated_time`)
PARTITION BY cityHash64(`#user_id`) % 64
ORDER BY `#user_id`
SETTINGS index_granularity = 8192 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```
- ClickHouse node2 table schema
```sql
clickhouse02 :) show create table demo.users_dim_local;

SHOW CREATE TABLE demo.users_dim_local

Query id: 58eea283-3761-4fd9-bf6a-3d10b03a5742

┌─statement──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ CREATE TABLE demo.users_dim_local
(
    `#user_id` String,
    `#device_id` String,
    `#created_time` UInt64,
    `#updated_time` UInt64,
    `sex` String,
    `user_name` String,
    `remark` String
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{uuid}/{all}', '{index}', `#updated_time`)
PARTITION BY cityHash64(`#user_id`) % 64
ORDER BY `#user_id`
SETTINGS index_granularity = 8192 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

ClickHouse node excepiton log:
- ClickHouse node1 log
```log
2022.11.03 07:06:44.957775 [ 48 ] {} <Debug> TCP-Session: 638d548e-250f-41a6-86af-0d6b042e7a92 Creating query context from session context, user_id: 94309d50-4f52-5250-31bd-74fecac179db, parent context user: default
2022.11.03 07:06:44.957920 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> executeQuery: (from 127.0.0.1:32888) ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String; (stage: Complete)
2022.11.03 07:06:44.957954 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Trace> ContextAccess (default): Access granted: ALTER ADD COLUMN(remark) ON demo.users_dim_local
2022.11.03 07:06:44.957977 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DatabaseReplicated (demo): Proposing query: ALTER TABLE users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:44.961935 [ 272 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:06:44.962000 [ 271 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:06:44.962437 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DDLWorker(demo): Waiting for worker thread to process all entries before query-0000000004
2022.11.03 07:06:44.962707 [ 272 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=0, last_current_task=none, last_skipped_entry_name=query-0000000003
2022.11.03 07:06:44.962732 [ 272 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:06:44.962741 [ 272 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:06:44.962836 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:06:44.963384 [ 272 ] {} <Trace> DDLWorker(demo): Waiting for initiator 01|clickhouse01 to commit or rollback entry /clickhouse/default/demo/log/query-0000000004
2022.11.03 07:06:44.964066 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:44.975145 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> executeQuery: (from 0.0.0.0:0, user: ) /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String (stage: Complete)
2022.11.03 07:06:44.979373 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Updated shared metadata nodes in ZooKeeper. Waiting for replicas to apply changes.
2022.11.03 07:06:44.979408 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Waiting for 0101 to process log entry
2022.11.03 07:06:44.979414 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Waiting for 0101 to pull log-0000000000 to queue
2022.11.03 07:06:44.980101 [ 272 ] {} <Debug> DDLWorker(demo): Will not execute task query-0000000004: Entry query-0000000004 has been executed as initial query
2022.11.03 07:06:44.980128 [ 272 ] {} <Debug> DDLWorker(demo): Waiting for queue updates
2022.11.03 07:06:44.981873 [ 79 ] {} <Debug> demo.users_dim_local (ReplicatedMergeTreeQueue): Pulling 1 entries to queue: log-0000000000 - log-0000000000
2022.11.03 07:06:44.984918 [ 79 ] {} <Trace> demo.users_dim_local (ReplicatedMergeTreeQueue): Adding alter metadata version 1 to the queue
2022.11.03 07:06:44.984938 [ 79 ] {} <Debug> demo.users_dim_local (ReplicatedMergeTreeQueue): Pulled 1 entries to queue.
2022.11.03 07:06:44.985075 [ 87 ] {} <Information> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Metadata changed in ZooKeeper. Applying changes locally.
2022.11.03 07:06:44.985897 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Looking for node corresponding to log-0000000000 in 0101 queue
2022.11.03 07:06:44.990708 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Waiting for queue-0000000000 to disappear from 0101 queue
2022.11.03 07:06:44.991220 [ 87 ] {} <Information> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Applied changes to the metadata of the table. Current metadata version: 1
2022.11.03 07:06:44.992740 [ 87 ] {} <Trace> demo.users_dim_local (ReplicatedMergeTreeQueue): Finishing metadata alter with version 1
2022.11.03 07:06:44.994492 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DDLWorker(demo): Executed query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:44.994513 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> DDLWorker(demo): Task query-0000000004 executed by current replica
2022.11.03 07:07:05.576740 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Error> executeQuery: Code: 49. DB::Exception: There was an error on 02|clickhouse02: Cannot execute replicated DDL query, maximum retries exceeded (probably it's a bug): While executing DDLQueryStatus. (LOGICAL_ERROR) (version 22.10.1.1877 (official build)) (from 127.0.0.1:32888) (in query: ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String;), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0xcee4fb3 in /usr/bin/clickhouse
2. DB::DDLQueryStatusSource::generate() @ 0x12973d9c in /usr/bin/clickhouse
3. DB::ISource::tryGenerate() @ 0x13590475 in /usr/bin/clickhouse
4. DB::ISource::work() @ 0x13590006 in /usr/bin/clickhouse
5. DB::ExecutionThreadContext::executeTask() @ 0x135ac186 in /usr/bin/clickhouse
6. DB::PipelineExecutor::executeStepImpl(unsigned long, std::__1::atomic<bool>*) @ 0x135a02dc in /usr/bin/clickhouse
7. DB::PipelineExecutor::executeImpl(unsigned long) @ 0x1359ecf9 in /usr/bin/clickhouse
8. DB::PipelineExecutor::execute(unsigned long) @ 0x1359eafd in /usr/bin/clickhouse
9. ? @ 0x135afae9 in /usr/bin/clickhouse
10. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
11. ? @ 0xceff7be in /usr/bin/clickhouse
12. ? @ 0x7f8b43a91609 in ?
13. clone @ 0x7f8b439b6133 in ?
2022.11.03 07:07:05.576805 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Error> TCPHandler: Code: 49. DB::Exception: There was an error on 02|clickhouse02: Cannot execute replicated DDL query, maximum retries exceeded (probably it's a bug): While executing DDLQueryStatus. (LOGICAL_ERROR), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0xcee4fb3 in /usr/bin/clickhouse
2. DB::DDLQueryStatusSource::generate() @ 0x12973d9c in /usr/bin/clickhouse
3. DB::ISource::tryGenerate() @ 0x13590475 in /usr/bin/clickhouse
4. DB::ISource::work() @ 0x13590006 in /usr/bin/clickhouse
5. DB::ExecutionThreadContext::executeTask() @ 0x135ac186 in /usr/bin/clickhouse
6. DB::PipelineExecutor::executeStepImpl(unsigned long, std::__1::atomic<bool>*) @ 0x135a02dc in /usr/bin/clickhouse
7. DB::PipelineExecutor::executeImpl(unsigned long) @ 0x1359ecf9 in /usr/bin/clickhouse
8. DB::PipelineExecutor::execute(unsigned long) @ 0x1359eafd in /usr/bin/clickhouse
9. ? @ 0x135afae9 in /usr/bin/clickhouse
10. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
11. ? @ 0xceff7be in /usr/bin/clickhouse
12. ? @ 0x7f8b43a91609 in ?
13. clone @ 0x7f8b439b6133 in ?
2022.11.03 07:07:05.576838 [ 48 ] {b13a746a-db89-405c-a4c3-7a0c9cc3f808} <Debug> TCPHandler: Processed in 20.619131789 sec.
```
- ClickHouse node2 log
```log
2022.11.03 07:06:44.961968 [ 268 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:06:44.962004 [ 237 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:06:44.962786 [ 268 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=1, last_current_task=query-0000000003, last_skipped_entry_name=query-0000000002
2022.11.03 07:06:44.962801 [ 268 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:06:44.962805 [ 268 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:06:44.963389 [ 268 ] {} <Trace> DDLWorker(demo): Waiting for initiator 01|clickhouse01 to commit or rollback entry /clickhouse/default/demo/log/query-0000000004
2022.11.03 07:06:44.981293 [ 268 ] {} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:06:44.981727 [ 139 ] {} <Debug> demo.users_dim_local (ReplicatedMergeTreeQueue): Pulling 1 entries to queue: log-0000000000 - log-0000000000
2022.11.03 07:06:44.982587 [ 268 ] {} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:44.984437 [ 139 ] {} <Trace> demo.users_dim_local (ReplicatedMergeTreeQueue): Adding alter metadata version 1 to the queue
2022.11.03 07:06:44.984460 [ 139 ] {} <Debug> demo.users_dim_local (ReplicatedMergeTreeQueue): Pulled 1 entries to queue.
2022.11.03 07:06:44.984652 [ 78 ] {} <Information> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Metadata changed in ZooKeeper. Applying changes locally.
2022.11.03 07:06:44.988439 [ 78 ] {} <Information> demo.users_dim_local (e9858577-d248-4382-92d9-3c31f995f1c5): Applied changes to the metadata of the table. Current metadata version: 1
2022.11.03 07:06:44.991996 [ 78 ] {} <Trace> demo.users_dim_local (ReplicatedMergeTreeQueue): Finishing metadata alter with version 1
2022.11.03 07:06:44.999687 [ 268 ] {1787dec8-8917-4f2e-9ee9-f382c8d0fd27} <Debug> executeQuery: (from 0.0.0.0:0, user: ) /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String (stage: Complete)
2022.11.03 07:06:45.000190 [ 268 ] {1787dec8-8917-4f2e-9ee9-f382c8d0fd27} <Error> executeQuery: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)) (from 0.0.0.0:0) (in query: /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
2022.11.03 07:06:45.000440 [ 268 ] {1787dec8-8917-4f2e-9ee9-f382c8d0fd27} <Error> DDLWorker(demo): Query ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String wasn't finished successfully: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:45.000511 [ 268 ] {} <Debug> DDLWorker(demo): Task query-0000000004 executed by current replica
2022.11.03 07:06:45.003468 [ 268 ] {} <Error> DDLWorker(demo): Unexpected error, will try to restart main thread: Code: 341. DB::Exception: Unexpected error: 15
Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)). (UNFINISHED), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0x7c42c58 in /usr/bin/clickhouse
2. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7df41 in /usr/bin/clickhouse
3. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
4. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
5. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
6. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
7. ? @ 0xceff7be in /usr/bin/clickhouse
8. ? @ 0x7f84d8a28609 in ?
9. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:45.003505 [ 268 ] {} <Information> DDLWorker(demo): Cleaned DDLWorker state
2022.11.03 07:06:50.006492 [ 268 ] {} <Debug> DDLWorker(demo): Initialized DDLWorker thread
2022.11.03 07:06:50.006539 [ 268 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:06:50.006547 [ 268 ] {} <Trace> DDLWorker(demo): Don't have unfinished tasks after restarting
2022.11.03 07:06:50.006567 [ 237 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:06:50.007006 [ 268 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=0, last_current_task=none, last_skipped_entry_name=query-0000000003
2022.11.03 07:06:50.007020 [ 268 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:06:50.007035 [ 268 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:06:50.008720 [ 268 ] {} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:06:50.009733 [ 268 ] {} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:50.017681 [ 268 ] {bfe75d45-fa8a-4e6b-89c1-c3e4cfc7c2d5} <Debug> executeQuery: (from 0.0.0.0:0, user: ) /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String (stage: Complete)
2022.11.03 07:06:50.017930 [ 268 ] {bfe75d45-fa8a-4e6b-89c1-c3e4cfc7c2d5} <Error> executeQuery: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)) (from 0.0.0.0:0) (in query: /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
2022.11.03 07:06:50.018043 [ 268 ] {bfe75d45-fa8a-4e6b-89c1-c3e4cfc7c2d5} <Error> DDLWorker(demo): Query ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String wasn't finished successfully: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:50.018080 [ 268 ] {} <Debug> DDLWorker(demo): Task query-0000000004 executed by current replica
2022.11.03 07:06:50.020169 [ 268 ] {} <Error> DDLWorker(demo): Unexpected error, will try to restart main thread: Code: 341. DB::Exception: Unexpected error: 15
Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)). (UNFINISHED), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0x7c42c58 in /usr/bin/clickhouse
2. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7df41 in /usr/bin/clickhouse
3. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
4. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
5. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
6. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
7. ? @ 0xceff7be in /usr/bin/clickhouse
8. ? @ 0x7f84d8a28609 in ?
9. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:50.020203 [ 268 ] {} <Information> DDLWorker(demo): Cleaned DDLWorker state
2022.11.03 07:06:55.022623 [ 268 ] {} <Debug> DDLWorker(demo): Initialized DDLWorker thread
2022.11.03 07:06:55.022737 [ 268 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:06:55.024180 [ 268 ] {} <Trace> DDLWorker(demo): Don't have unfinished tasks after restarting
2022.11.03 07:06:55.022802 [ 237 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:06:55.025521 [ 268 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=0, last_current_task=none, last_skipped_entry_name=query-0000000003
2022.11.03 07:06:55.025545 [ 268 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:06:55.025550 [ 268 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:06:55.027817 [ 268 ] {} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:06:55.029319 [ 268 ] {} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:06:55.036934 [ 268 ] {ce997f21-4233-44e4-8e84-3af95fc37fc8} <Debug> executeQuery: (from 0.0.0.0:0, user: ) /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String (stage: Complete)
2022.11.03 07:06:55.037191 [ 268 ] {ce997f21-4233-44e4-8e84-3af95fc37fc8} <Error> executeQuery: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)) (from 0.0.0.0:0) (in query: /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
2022.11.03 07:06:55.037318 [ 268 ] {ce997f21-4233-44e4-8e84-3af95fc37fc8} <Error> DDLWorker(demo): Query ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String wasn't finished successfully: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:55.037352 [ 268 ] {} <Debug> DDLWorker(demo): Task query-0000000004 executed by current replica
2022.11.03 07:06:55.039580 [ 268 ] {} <Error> DDLWorker(demo): Unexpected error, will try to restart main thread: Code: 341. DB::Exception: Unexpected error: 15
Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)). (UNFINISHED), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0x7c42c58 in /usr/bin/clickhouse
2. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7df41 in /usr/bin/clickhouse
3. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
4. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
5. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
6. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
7. ? @ 0xceff7be in /usr/bin/clickhouse
8. ? @ 0x7f84d8a28609 in ?
9. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:06:55.039608 [ 268 ] {} <Information> DDLWorker(demo): Cleaned DDLWorker state
2022.11.03 07:07:00.044224 [ 268 ] {} <Debug> DDLWorker(demo): Initialized DDLWorker thread
2022.11.03 07:07:00.044357 [ 268 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:07:00.044374 [ 268 ] {} <Trace> DDLWorker(demo): Don't have unfinished tasks after restarting
2022.11.03 07:07:00.044426 [ 237 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:07:00.045160 [ 268 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=0, last_current_task=none, last_skipped_entry_name=query-0000000003
2022.11.03 07:07:00.045174 [ 268 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:07:00.045185 [ 268 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:07:00.046979 [ 268 ] {} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:07:00.049145 [ 268 ] {} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:07:00.057805 [ 268 ] {87c78b92-67e0-4b5e-826d-14b013c6861b} <Debug> executeQuery: (from 0.0.0.0:0, user: ) /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String (stage: Complete)
2022.11.03 07:07:00.058047 [ 268 ] {87c78b92-67e0-4b5e-826d-14b013c6861b} <Error> executeQuery: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)) (from 0.0.0.0:0) (in query: /* ddl_entry=query-0000000004 */ ALTER TABLE demo.users_dim_local ADD COLUMN `remark` String), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
2022.11.03 07:07:00.058163 [ 268 ] {87c78b92-67e0-4b5e-826d-14b013c6861b} <Error> DDLWorker(demo): Query ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String wasn't finished successfully: Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. DB::AlterCommands::validate(std::__1::shared_ptr<DB::IStorage> const&, std::__1::shared_ptr<DB::Context const>) const @ 0x12b1c482 in /usr/bin/clickhouse
2. DB::InterpreterAlterQuery::executeToTable(DB::ASTAlterQuery const&) @ 0x124a670d in /usr/bin/clickhouse
3. DB::InterpreterAlterQuery::execute() @ 0x124a4032 in /usr/bin/clickhouse
4. ? @ 0x1297956b in /usr/bin/clickhouse
5. DB::executeQuery(DB::ReadBuffer&, DB::WriteBuffer&, bool, std::__1::shared_ptr<DB::Context>, std::__1::function<void (std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>, std::__1::optional<DB::FormatSettings> const&) @ 0x1297e52e in /usr/bin/clickhouse
6. DB::DDLWorker::tryExecuteQuery(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7f268 in /usr/bin/clickhouse
7. DB::DDLWorker::tryExecuteQueryOnLeaderReplica(DB::DDLTaskBase&, std::__1::shared_ptr<DB::IStorage>, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::shared_ptr<zkutil::ZooKeeper> const&, std::__1::unique_ptr<zkutil::ZooKeeperLock, std::__1::default_delete<zkutil::ZooKeeperLock> >&) @ 0x11f82179 in /usr/bin/clickhouse
8. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7d90b in /usr/bin/clickhouse
9. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
10. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
11. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
12. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
13. ? @ 0xceff7be in /usr/bin/clickhouse
14. ? @ 0x7f84d8a28609 in ?
15. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:07:00.058197 [ 268 ] {} <Debug> DDLWorker(demo): Task query-0000000004 executed by current replica
2022.11.03 07:07:00.060434 [ 268 ] {} <Error> DDLWorker(demo): Unexpected error, will try to restart main thread: Code: 341. DB::Exception: Unexpected error: 15
Code: 15. DB::Exception: Cannot add column `remark`: column with this name already exists. (DUPLICATE_COLUMN) (version 22.10.1.1877 (official build)). (UNFINISHED), Stack trace (when copying this message, always include the lines below):
0. DB::Exception::Exception(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, int, bool) @ 0xce3f35a in /usr/bin/clickhouse
1. ? @ 0x7c42c58 in /usr/bin/clickhouse
2. DB::DDLWorker::processTask(DB::DDLTaskBase&, std::__1::shared_ptr<zkutil::ZooKeeper> const&) @ 0x11f7df41 in /usr/bin/clickhouse
3. DB::DDLWorker::scheduleTasks(bool) @ 0x11f7b373 in /usr/bin/clickhouse
4. DB::DDLWorker::runMainThread() @ 0x11f749f2 in /usr/bin/clickhouse
5. void std::__1::__function::__policy_invoker<void ()>::__call_impl<std::__1::__function::__default_alloc_func<ThreadFromGlobalPoolImpl<true>::ThreadFromGlobalPoolImpl<void (DB::DDLWorker::*)(), DB::DDLWorker*>(void (DB::DDLWorker::*&&)(), DB::DDLWorker*&&)::'lambda'(), void ()> >(std::__1::__function::__policy_storage const*) @ 0x11f89aa7 in /usr/bin/clickhouse
6. ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) @ 0xcefa04c in /usr/bin/clickhouse
7. ? @ 0xceff7be in /usr/bin/clickhouse
8. ? @ 0x7f84d8a28609 in ?
9. clone @ 0x7f84d894d133 in ?
 (version 22.10.1.1877 (official build))
2022.11.03 07:07:00.060463 [ 268 ] {} <Information> DDLWorker(demo): Cleaned DDLWorker state
2022.11.03 07:07:05.066977 [ 268 ] {} <Debug> DDLWorker(demo): Initialized DDLWorker thread
2022.11.03 07:07:05.067055 [ 237 ] {} <Trace> DDLWorker(demo): Too early to clean queue, will do it later.
2022.11.03 07:07:05.067071 [ 268 ] {} <Debug> DDLWorker(demo): Scheduling tasks
2022.11.03 07:07:05.068162 [ 268 ] {} <Trace> DDLWorker(demo): Don't have unfinished tasks after restarting
2022.11.03 07:07:05.068812 [ 268 ] {} <Trace> DDLWorker(demo): scheduleTasks: initialized=true, size_before_filtering=4, queue_size=4, entries=query-0000000001..query-0000000004, first_failed_task_name=none, current_tasks_size=0, last_current_task=none, last_skipped_entry_name=query-0000000003
2022.11.03 07:07:05.068832 [ 268 ] {} <Debug> DDLWorker(demo): Will schedule 1 tasks starting from query-0000000004
2022.11.03 07:07:05.068838 [ 268 ] {} <Trace> DDLWorker(demo): Checking task query-0000000004
2022.11.03 07:07:05.070881 [ 268 ] {} <Debug> DDLWorker(demo): Processing task query-0000000004 (ALTER TABLE users_dim_local
    ADD COLUMN `remark` String)
2022.11.03 07:07:05.071841 [ 268 ] {} <Debug> DDLWorker(demo): Executing query: ALTER TABLE demo.users_dim_local
    ADD COLUMN `remark` String
2022.11.03 07:07:05.079836 [ 268 ] {} <Warning> DDLWorker(demo): Task query-0000000004 was not executed by anyone, maximum number of retries exceeded
2022.11.03 07:07:05.082350 [ 268 ] {} <Debug> DDLWorker(demo): Waiting for queue updates
```

### Drop column case
- Alter table drop column
```sql
clickhouse01 :) ALTER TABLE demo.users_dim_local  DROP COLUMN `remark`;

ALTER TABLE demo.users_dim_local
    DROP COLUMN remark

Query id: cfc40bc0-1dfa-4641-9e12-98df59a6b9b5

┌─shard─┬─replica──────┬─status─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ 01    │ clickhouse01 │ OK     │                   1 │                0 │
└───────┴──────────────┴────────┴─────────────────────┴──────────────────┘
↙ Progress: 1.00 rows, 49.00 B (0.05 rows/s., 2.38 B/s.)  49%
1 row in set. Elapsed: 20.598 sec.

Received exception from server (version 22.10.1):
Code: 49. DB::Exception: Received from localhost:9000. DB::Exception: There was an error on 02|clickhouse02: Cannot execute replicated DDL query, maximum retries exceeded (probably it's a bug): While executing DDLQueryStatus. (LOGICAL_ERROR)
```

Table schema after "Alter table drop column" query execution, actually, clickhouse 2 nodes successfully dropped `remark` column.
- ClickHouse node1 table schema
```sql
clickhouse01 :) show create table demo.users_dim_local;

SHOW CREATE TABLE demo.users_dim_local

Query id: 0a986826-1a60-4d43-97a4-72370ed3562f

┌─statement──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ CREATE TABLE demo.users_dim_local
(
    `#user_id` String,
    `#device_id` String,
    `#created_time` UInt64,
    `#updated_time` UInt64,
    `sex` String,
    `user_name` String
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{uuid}/{all}', '{index}', `#updated_time`)
PARTITION BY cityHash64(`#user_id`) % 64
ORDER BY `#user_id`
SETTINGS index_granularity = 8192 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

- ClickHouse node2 table schema
```sql
clickhouse02 :) show create table demo.users_dim_local;

SHOW CREATE TABLE demo.users_dim_local

Query id: 66bfd485-00e8-4db0-b2e0-8174e411865f

┌─statement──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ CREATE TABLE demo.users_dim_local
(
    `#user_id` String,
    `#device_id` String,
    `#created_time` UInt64,
    `#updated_time` UInt64,
    `sex` String,
    `user_name` String
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{uuid}/{all}', '{index}', `#updated_time`)
PARTITION BY cityHash64(`#user_id`) % 64
ORDER BY `#user_id`
SETTINGS index_granularity = 8192 │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```