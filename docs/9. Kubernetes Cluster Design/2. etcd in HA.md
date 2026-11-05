# etcd in HA

etcd is a distributed, reliable key-value store. In an HA cluster, etcd runs on multiple servers, and each server keeps an identical copy of the data. If one server is lost, the other copies still hold the data.

## Consistency and leader election

Reads can be served by any etcd instance, because every instance has the same data. Writes are different. etcd does not process writes on every node. Instead, the cluster elects one leader, and the leader processes all writes:

- If a write arrives at the leader, the leader processes it.
- If a write arrives at a follower, the follower forwards it to the leader.
- The leader replicates the write to the other members. A write is complete only after the other members confirm it.

etcd uses the Raft consensus protocol to elect the leader and to replicate writes. If the followers stop receiving heartbeats from the leader, because the leader failed or lost network connectivity, they start a new election and elect a new leader.

## Quorum

A write is complete when a majority of the members store it. This majority is called the **quorum**:

```
quorum = floor(n / 2) + 1
```

where `n` is the number of etcd members. For example:

- 3 members: quorum is 2, so the cluster tolerates the loss of 1 member.
- 5 members: quorum is 3, so the cluster tolerates the loss of 2 members.

If the cluster does not have quorum, it cannot process writes.

## Number of members

- One or two members do not provide HA. Losing one member leaves the cluster without quorum.
- Use an odd number of members. An even number does not add fault tolerance and can leave the cluster without quorum during a network partition.
- Three members is the minimum for HA. Five members give higher fault tolerance. More than five members adds cost without much benefit.
