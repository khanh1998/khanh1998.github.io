---
date: '2026-06-28T12:47:02+07:00'
draft: false
title: 'Postgres Internal Insert Wait'
---

I observe the pg_stat_activity during high insert load and saw some wait event:
- LWLock:BufferContent
- LWLock:WalInsert
- LWLock:WalWrite
- IO:WalWrite
- IO:WalSync
- Lock:extend
- IO:DataFileExtend


## LWLock:BufferContent
This usually happen under high concurrency insert on a high vcpu instance.
If primary key of an high concurrency insert table using increment integer, you will probably see this wait event.
It is because when applying increment integer to a Btree index, whenever you insert a new row, the primary key get incremented by one, the new primary key will be append to the right most leaf of the Btree index.
If the number of connections and vcpu is low, meaning low concurrency, you probably won't see this. But when number of connection is high, vcpu is hight, the insert load is high, that means many connections will concurrently acquire an exclusive lock on the right most leaf page of the primary key index - causing the locking contention.
You can change the primary key to a random value like uuid v4, the wait LWLock:BufferContent will be disappear. Because the new primary key values are random, it don't specifically target the right most leaf, but will distribute the load balance between the leafs.
But uuid v4 is not all good. it good trade offs.