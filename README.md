# ScyllaDB Performance Evaluation

This evaluation compares two 30-minute mixed workloads on a three-node ScyllaDB 2026.3.6 cluster on AWS. Each node used local NVMe with XFS. A separate loader ran scylla-bench, and a separate monitoring host collected ScyllaDB and OS metrics with Prometheus and Grafana.

## Benchmark setup

- Preload of 100 million rows with 1 KiB values, totaling 102.4 GB of logical value data before replication and overhead.
- RF3 and QUORUM consistency in a single availability zone.
- Concurrent uniform reads and writes with rate caps of 30,000 reads/s and 10,000 writes/s.
- Both trials wrote across all 100,000 partitions. The baseline also read across all 100,000 partitions. The follow-up read across 100 warmed partitions while keeping the full dataset in place.

## Results

| Read range | Reads/s | Raw read p99 | Writes/s | Raw write p99 |
| --- | ---: | ---: | ---: | ---: |
| Full range | 3,560 | 250.74 ms | 9,999 | 2.42 ms |
| Warmed subset | 29,999 | 4.39 ms | 9,999 | 3.54 ms |


Both workloads reported zero client errors in both runs. Archived Prometheus counters showed foreground read I/O dropping from about 36,629/s to 2.6/s in the warmed trial. Compaction I/O continued in both runs. The full-range baseline missed the read throughput and latency goals. The follow-up met both goals under the tested access locality.

The two workloads represent different read distributions and starting cache conditions. Use a representative key distribution, working set, and failure scenario when validating production capacity.
