---
layout: post
title:  "Dead Lock Log Analysis"
author: "YaoFly"
tags: ["Java", "mysql", "lock"]
---   

这是开发环境里出现的一份死锁日志：

```
=====================================
2021-01-22 11:20:09 0x7fdfe0806700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 18 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 113420 srv_active, 0 srv_shutdown, 221777 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 66855
OS WAIT ARRAY INFO: signal count 351264
RW-shared spins 1716325, rounds 1719356, OS waits 1869
RW-excl spins 347700, rounds 578644, OS waits 3230
RW-sx spins 205, rounds 3070, OS waits 68
Spin rounds per wait: 1.00 RW-shared, 1.66 RW-excl, 14.98 RW-sx
------------------------
LATEST DETECTED DEADLOCK
------------------------
2021-01-18 19:52:12 0x7fdfee4f6700
*** (1) TRANSACTION:
TRANSACTION 4071457, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 1912, OS thread handle 140599551682304, query id 159296 10.244.0.0 root executing
SELECT  id,organization_code,year,function_module,workflow_name,workflow_no,approve_status,approver,approve_start_time,approve_end_time,   
process_definition_id,process_id,audit_status,process_status,remark,create_time,create_user,update_time,update_user     
FROM t_finance_sys_version_record WHERE organization_code = 'ORG1334336382416412673' AND year = 2021 AND function_module = 1 AND process_status = 3    
ORDER BY create_time DESC limit 1 for update

*** (1) HOLDS THE LOCK(S):
/**
 * 事务一是一条带有for update 的select语句
 * lock_mode X waiting Record lock
 * 可以看到ID为4071457的事务在等待`t_finance_sys_version_record`表中,索引名为idx_version_record_orgid的索引上堆物理编号为10的记录锁
 */
RECORD LOCKS space id 2903 page no 5 n bits 112 index idx_version_record_orgid of table `vv_finance`.`t_finance_sys_version_record` trx id 4071457 lock_mode X waiting
Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 373934343165663863633031666535363463303433326135343264383961; asc  ; (total 32 bytes);


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 2903 page no 5 n bits 112 index idx_version_record_orgid of table `vv_finance`.`t_finance_sys_version_record` trx id 4071457 lock_mode X waiting
Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 373934343165663863633031666535363463303433326135343264383961; asc 79441ef8cc01fe564c0432a542d89a; (total 32 bytes);


*** (2) TRANSACTION:
TRANSACTION 4071456, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 12 row lock(s), undo log entries 1
MySQL thread id 1917, OS thread handle 140599520356096, query id 159298 10.244.0.0 root update
INSERT INTO t_finance_sys_version_record  ( id,
organization_code,
year,
function_module,
process_status,
create_time,
create_user,
update_time,
update_user )  VALUES  ( '33e1e4ae56343e46fc4495f7ae8b495e',
'ORG1334336382416412673',
2021,
1,
3,
'2021-01-18 19:52:12.376',
'oa1226736233564770306',
'2021-01-18 19:52:12.376',
'oa1226736233564770306' )

*** (2) HOLDS THE LOCK(S):
/*
 * 事务二是 一条与事务一同样的for update 查询语句与一条Insert语句
 * lock_mode X Record lock
 * 由于事务二先执行了for update查询，可以看到当前事务持有了索引名为idx_version_record_orgid的索引上值为ORG1334336382416412673的记录锁，锁住了堆物理编号为2,5,10,11,40的行记录
 * 这样就很清楚的知道事务一所申请的记录锁由事务二所持有
 */
RECORD LOCKS space id 2903 page no 5 n bits 112 index idx_version_record_orgid of table `vv_finance`.`t_finance_sys_version_record` trx id 4071456 lock_mode X
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 666538343864336638623536343532646130653861613435313539663361; asc fe848d3f8b56452da0e8aa45159f3a; (total 32 bytes);

Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 656637363835616237616637373563613839633331343661363833656638; asc ef7685ab7af775ca89c3146a683ef8; (total 32 bytes);

Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 373934343165663863633031666535363463303433326135343264383961; asc 79441ef8cc01fe564c0432a542d89a; (total 32 bytes);

Record lock, heap no 11 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 386433306133323033336534616432653933653732393165366161353535; asc 8d30a32033e4ad2e93e7291e6aa555; (total 32 bytes);

Record lock, heap no 40 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 663531376162623931326332343062636265336537353438333932316436; asc f517abb912c240bcbe3e75483921d6; (total 32 bytes);


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
/*
 * lock_mode X locks gap before rec insert intention waiting Record lock
 * 可以看出来事务二正在申请heap no 10 PHYSICAL RECORD堆物理编号10的插入意向锁（insert intention lock）。   
   为当前数据库事务等级为RR，所以数据库在进行插入操作时，会申请要插入数据周围的间隙锁，防止产生幻读。注意此时插入意向锁是一种特殊的间隙锁。   
 * 事务二在申请到heap no 10 的记录锁之后，事务一也排队申请此记录锁，而事务二对同一条记录间隙锁的申请排队排在了事务一之后，导致了死锁的发生。
 */
RECORD LOCKS space id 2903 page no 5 n bits 112 index idx_version_record_orgid of table `vv_finance`.`t_finance_sys_version_record` trx id 4071456 
lock_mode X locks gap before rec insert intention waiting Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 22; hex 4f524731333334333336333832343136343132363733; asc ORG1334336382416412673;;
 1: len 30; hex 373934343165663863633031666535363463303433326135343264383961; asc 79441ef8cc01fe564c0432a542d89a; (total 32 bytes);

/*
 * Mysql 最后采取了回滚事务一来解决此次死锁，并抛出死锁异常。
 */
*** WE ROLL BACK TRANSACTION (1)
------------
TRANSACTIONS
------------
```
