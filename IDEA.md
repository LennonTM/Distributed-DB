# RivetDB — Engineering Specification

## Overview

RivetDB is a from-scratch, replicated key-value database written in **Python**.

It is a learning project rather than a production database. The goal is to build a small but realistic distributed database that remains correct through crashes, restarts, leader changes, packet loss, and network partitions.

The finished project should demonstrate:

- durable local storage;
- crash recovery;
- an LSM-tree storage engine;
- write-ahead logging;
- SSTables and compaction;
- Bloom filters and checksums;
- Raft consensus;
- replicated writes;
- quorum-based commits;
- snapshots and stale-node recovery;
- network partition handling;
- linearizable reads and writes;
- deterministic fault testing;
- chaos testing;
- verification that acknowledged writes are never lost.

---

## Product

RivetDB is a **single-shard, strongly consistent distributed key-value store**.

A cluster consists of **3 or 5 nodes**.

Every node stores the complete dataset.

One node acts as the Raft leader. Clients may connect to any node, but writes and linearizable reads are coordinated through the leader.

Supported operations:

```text
PUT key value
GET key
DELETE key
STATUS
```

Optional later:

```text
CAS key expected new_value
```

---

## Language

The project should be implemented primarily in:

```text
Python 3.13+
```

Libraries may be used for general infrastructure such as networking, serialization, logging, CLI handling, and testing.

The following should be implemented as part of the project rather than delegated to existing database or consensus libraries:

- storage engine;
- WAL;
- SSTables;
- Bloom filter;
- compaction;
- Raft;
- snapshots;
- failure simulation.

---

## Correctness Requirements

The primary guarantee is:

> Once RivetDB acknowledges a write as committed, that write must not be lost after any supported failure from which the cluster can eventually recover a quorum.

The system must also guarantee that:

- successful writes are acknowledged only after Raft commits them;
- followers do not claim to have replicated data before it is durable;
- committed Raft entries are never overwritten;
- commands are applied in Raft log order;
- crashes cannot create data that was never durably written;
- corrupt or incomplete records are detected;
- final `GET`, `PUT`, and `DELETE` behavior is linearizable.

Correctness takes priority over performance.

---

## Storage

Each node should contain a custom LSM-tree based storage engine.

It should support:

- write-ahead logging;
- an in-memory memtable;
- immutable memtables;
- SSTables;
- Bloom filters;
- tombstones;
- compaction;
- persistent storage metadata;
- crash recovery.

The storage engine should survive abrupt process termination without losing writes that had already been acknowledged as durable.

Compaction and metadata updates must also be crash-safe.

---

## Raft

RivetDB should implement Raft directly.

Required functionality:

- follower, candidate, and leader states;
- randomized leader elections;
- voting;
- heartbeats;
- replicated log entries;
- quorum commit;
- leader failover;
- log conflict resolution;
- persistent Raft state;
- snapshots;
- snapshot installation;
- lagging follower recovery.

The Raft core should be testable independently from real networking so elections, partitions, stale messages, and failure cases can be reproduced deterministically.

---

## Reads

The final system must provide linearizable reads.

Reading from a node simply because it believes it is leader is not sufficient.

A leader must confirm that it still has authority before serving a linearizable read and must ensure its local state has applied all required committed entries.

---

## Networking

Nodes communicate over TCP.

The protocol should support:

- client-to-node communication;
- node-to-node Raft communication;
- message framing;
- protocol versioning;
- request/message IDs;
- checksums;
- partial reads and writes;
- reconnecting peers;
- reasonable message-size limits.

The system must tolerate:

- delayed messages;
- dropped messages;
- duplicate messages;
- stale messages;
- temporary partitions.

---

## Recovery

The following scenarios must be supported:

### Clean restart

A node reloads its durable state and rejoins the cluster.

### Abrupt crash

A node may be terminated without executing shutdown logic.

Previously acknowledged writes must survive.

### Stale follower

A follower that has missed a large number of updates should catch up using either retained Raft entries or a snapshot.

### Lost follower state

A follower whose local state is deleted should be able to recover completely from the cluster.

### Corrupt or incomplete data

Recovery must detect truncated or corrupt storage records rather than silently treating them as valid data.

---

## Network Partitions

Partition behavior must follow Raft quorum semantics.

For example, in a five-node cluster split as:

```text
[A, B] | [C, D, E]
```

the expected behavior is:

- the two-node side cannot commit new writes;
- the three-node side can elect a leader and continue committing;
- once the partition heals, conflicting uncommitted entries are resolved;
- previously committed entries remain intact.

---

## Failure Testing

The project should include deterministic fault injection for scenarios such as:

```text
drop A -> B
delay A -> C
drop a percentage of messages
duplicate messages
partition nodes
pause a node
heal the cluster
```

Randomized fault tests must use recorded seeds so failures can be reproduced.

The project must also test abrupt process termination and restart.

---

## Durability Verification

Chaos tests should continuously perform writes while nodes are killed, restarted, delayed, or partitioned.

Every successfully acknowledged write should also be recorded in an external verifier log that is independent of RivetDB.

After fault injection stops:

1. restore enough nodes to form a quorum;
2. allow the cluster to converge;
3. read every acknowledged write;
4. compare the stored value with the verifier history;
5. fail the test if any acknowledged write is missing or incorrect.

This external verifier is the evidence for the project's main durability guarantee.

---

## Consistency Testing

The test suite should include:

### Deterministic Raft tests

Verify properties such as:

- no more than one leader per term;
- committed entries never change;
- applied entries are committed;
- application order never moves backwards;
- leader commit follows quorum rules;
- Raft log consistency is preserved.

### Black-box history tests

Record concurrent client operations and verify that observed behavior is linearizable.

A small custom linearizability checker should be part of the project.

---

## Scope

### In scope

- one Raft group;
- static membership;
- 3- and 5-node clusters;
- TCP networking;
- custom storage engine;
- WAL;
- SSTables;
- Bloom filters;
- compaction;
- snapshots;
- crash recovery;
- node catch-up;
- network partitions;
- deterministic failure simulation;
- chaos testing;
- linearizability testing;
- benchmarks.

### Out of scope

- sharding;
- dynamic membership;
- distributed transactions;
- SQL;
- secondary indexes;
- authentication;
- authorization;
- TLS;
- Kubernetes;
- cloud deployment;
- automatic rebalancing;
- production-grade observability;
- Byzantine failures.

---

## Milestones

### 1. Single-node durable store

Implement:

- `PUT`;
- `GET`;
- `DELETE`;
- networking;
- WAL;
- memtable;
- restart recovery.

Success condition:

> Kill the process after acknowledged writes, restart it, and recover all of them.

### 2. SSTables

Add:

- immutable memtables;
- SSTables;
- indexes;
- Bloom filters;
- storage metadata.

Success condition:

> Data survives repeated flushes and restarts.

### 3. Compaction

Add:

- SSTable merging;
- tombstones;
- crash-safe compaction.

Success condition:

> Compaction does not change the visible contents of the database.

### 4. In-memory Raft

Add:

- elections;
- voting;
- heartbeats;
- replication;
- quorum commit;
- deterministic simulation.

Success condition:

> Election and partition tests pass without using real sockets or processes.

### 5. Persistent replicated cluster

Connect Raft, networking, and storage.

Success condition:

> A three-node cluster continues after one node dies and does not lose acknowledged writes.

### 6. Snapshots and stale-node recovery

Add:

- snapshots;
- Raft log truncation;
- snapshot transfer;
- follower restoration.

Success condition:

> Delete a follower's state and allow it to rebuild itself from the cluster.

### 7. Partition and consistency testing

Add:

- partition simulation;
- leader isolation tests;
- linearizable reads;
- history recording;
- linearizability checking.

Success condition:

> Correct runs pass and deliberately introduced consistency bugs are detected.

### 8. Chaos testing

Add repeated:

- node killing;
- restarting;
- network partitions;
- delays;
- message drops.

Success condition:

> A long seeded run finishes with zero lost acknowledged writes.

---

## Definition of Done

RivetDB v1 is complete when it demonstrates:

- durable single-node storage;
- crash recovery;
- SSTables and compaction;
- custom Raft consensus;
- replicated durable writes;
- quorum-based commit semantics;
- leader failover;
- linearizable reads;
- snapshots;
- stale-node recovery;
- network partition handling;
- deterministic failure tests;
- multi-process chaos tests;
- external verification of acknowledged writes;
- linearizability testing;
- reproducible benchmarks.

The final demonstration should run a five-node cluster under repeated node failures while clients continue writing.

After recovery, the verifier should report:

```text
cluster converged:           yes
lost acknowledged writes:    0
value mismatches:            0
linearizability violations:  0
result:                      PASS
```

The result should be based on externally recorded client history rather than the database simply reporting that it passed.