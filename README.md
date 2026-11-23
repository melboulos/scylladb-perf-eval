# ScyllaDB Performance Evaluation

## Challenge

Could a three-node ScyllaDB cluster sustain 30,000 reads/s and 10,000 writes/s
concurrently while keeping client p99 latency under 10 ms?

I preloaded 100 million rows with 1 KiB values, or 102.4 GB of logical value
data before replication and overhead. Both mixed runs lasted 30 minutes and
used RF3 and QUORUM on the same AWS cluster with local NVMe.

## Before — reads across the full dataset

With reads and writes spread across all 100,000 partitions, writes held
their target. Reads averaged 3,560/s at 250.74 ms p99 and degraded over the run.

## Investigation — where reads waited

I replayed the Prometheus archive and compared the first and last five
minutes. Mean foreground I/O queue wait rose from 10.33 ms to 32.20 ms
per I/O, while mean execution time stayed around 0.15 ms. Across the run,
97.69% of database-side cache reads needed data from SSTables.

## After — reads across a warmed subset

I kept the full dataset and write workload in place. The reader sampled
100 warmed partitions instead of 100,000. Row size, replication, consistency,
and the requested read and write rates stayed the same.

| Read workload | Achieved reads/s | Read p99 | Achieved writes/s | Write p99 |
| --- | ---: | ---: | ---: | ---: |
| Full range | 3,560 | 250.74 ms | 9,999 | 2.42 ms |
| Warmed subset | 29,999 | 4.39 ms | 9,999 | 3.54 ms |

Latency figures are raw client p99. Both runs reported zero client errors.
Foreground disk reads fell from about 36,629/s to 2.6/s in the warmed run,
even as compaction I/O increased.

## Takeaway

On the same hardware, the warmed workload met the throughput and p99
targets. The full-range workload missed the read targets. Read locality
made a material difference in how much work reached the disk, so I'd
measure the real key distribution and working set before using these
results to size a production cluster.
