---
layout: default
title: Surviving primary failure without losing a write in OpenSearch
---

# Surviving primary failure without losing a write in OpenSearch

## The problem

A primary dies. Writes keep arriving. Where do they go?

That is the entire problem of failover. The cluster must pick a new primary, bring it fully up to date, and redirect the write stream, all while the application keeps writing as if nothing happened. Any gap means lost writes. Any overlap means two primaries and split brain.

I was a core developer on OpenSearch segment replication. I did not design it. I helped build it, and failover is where I spent my time.

## Why failover is harder here

Under document replication, every copy indexes every write. A replica is almost a primary already. Promoting one is mostly a label change plus a short translog replay.

Segment replication breaks that symmetry. Replicas never index. They hold segment files copied from the primary, plus a translog. The replica closest to the old primary is still behind by definition: it is missing whatever the primary indexed since the last checkpoint.

So promotion has real work to do. The new primary must close that gap exactly, down to the last operation, before it takes a single write. Close is not enough. Exact is the requirement.

## The catch-up: phases 1 and 2

When the cluster manager picks the new primary, it chooses the replica with the highest replication checkpoint, the copy furthest ahead. That was one of my contributions: teach the allocator to prefer that replica, so the gap to close is as small as possible.

Recovery then runs in phases. Phase 1 copies any missing segment files. Phase 2 replays the translog tail, the operations that arrived after the last checkpoint. At the end of phase 2, the new primary holds everything the old one had.

Almost. There is a catch, and it is the heart of this story.

## The write queue

Writes did not stop during recovery. The application kept sending them. After phase 2, the system faces a choice: let new writes flow to a primary that is about to change, or hold them.

It holds them. Post phase 2, before the handoff, writes block. They queue. The new primary does a forced refresh from the old primary's latest state to catch up with every other replica, and only then does the handoff happen. The queue is the price of exactness: no write is accepted anywhere until exactly one node is ready to own all of them.

This is also the known cost. During primary relocation, write throughput can drop to zero for the duration of that forced catch-up. The cluster trades availability of writes for correctness of promotion. There is active work, a segment counter heuristic, to shrink that window by letting the new primary start its own engine earlier without file conflicts. But the queue itself is not a bug. It is the mechanism.

## The atomic handoff

Then the promotion, and it must be atomic.

One cluster state update flips the roles. Before it, the old primary is primary. After it, the new one is. There is no moment with two primaries accepting writes, and no moment with zero. Every node in the cluster applies the same state version, so every node agrees on who the primary is at every instant.

Atomicity is what makes the queue safe to drain. The queued writes were held precisely because no node was allowed to accept them yet. The moment promotion commits, exactly one node may, and the queue flows into it in order. No write is lost, none is duplicated, none lands on the wrong primary.

Think of it as changing the engine while the car runs. The queue is the clutch: disengage, shift, re-engage. The atomic handoff is the shift itself, one motion, no grinding.

## The trade-offs

The bill for exactness:

- Write stalls. Relocation can pin write throughput at zero while the new primary catches up. For planned relocations during scaling, this is the dominant cost.
- Replication lag matters more. The further the chosen replica trails, the longer the catch-up, the longer the queue waits. Picking the furthest-ahead replica is not an optimization. It shortens the stall.
- No partial promotion. The all-or-nothing handoff means a slow catch-up delays everything instead of degrading gracefully.

## Where I worked on it

Failover selection. I updated the primary shard allocator to prefer the replica with the highest replication checkpoint when promoting a new primary. Furthest ahead wins, translog replay is minimal, the queue waits less.

Primary balancing, the subject of my other article. Node-to-node replication concentrates network load on primaries, so the failover story only works if primaries stay spread evenly in the first place.

Replication resilience. I made replication survive local file read errors instead of failing the whole replication event, because a failover that trips on a corrupt read is not a failover.

## What I took with me

Failover is a consensus problem wearing a storage costume. The segments and checkpoints are details. The real work is the same as any leadership change: agree on exactly one leader, agree on exactly when, and park the work until the agreement is final. The write queue and the atomic handoff are just those three agreements made concrete.

---

*References: [PrimaryShardAllocator checkpoint preference](https://github.com/opensearch-project/OpenSearch/pull/4041), [primary balance constraint](https://github.com/opensearch-project/OpenSearch/pull/6422), [peer recovery for segment replication](https://github.com/opensearch-project/OpenSearch/pull/5344), [RFC on avoiding write blocking during relocation](https://github.com/opensearch-project/OpenSearch/issues/18355). Related: [Preventing hot shards and increasing throughput in OpenSearch](./allocator.html).*
