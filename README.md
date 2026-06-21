# RowKeyDB — Single-Node Beta

RowKeyDB is a database that speaks the Google Bigtable API and that you run on your own infrastructure, on any cloud or on bare metal. This repository holds its first single-node beta, built for testing and evaluation, together with the first benchmark numbers and the means to confirm that a download came from us.

It is, to my knowledge, the only Bigtable-compatible database available outside Google Cloud. It is written in careful, modern C++, with no garbage collector to make its latency unpredictable, and it is built to run on modern replicated disks — Google's pd-ssd, AWS's io2, Azure's Ultra Disk, and NVMe-over-Fabrics — so it needs no HDFS underneath it. It is a drop-in replacement for Google Bigtable and for HBase, which it serves through Google's HBase-to-Bigtable adapter, and it spares the operator the failures and the constant tuning that HBase has long been faulted for.

The single-node beta is not a toy. It is designed and tested never to lose an acknowledged write. When the server answers that a write has succeeded, that write has been flushed to disk and will survive an unannounced kill of the process. That guarantee is the whole point of this first release, and every number below was measured with it in force.

On a single machine, RowKeyDB serves **30,000 reads per second at a ninety-ninth-percentile latency of five milliseconds**, and it reaches **40,000 reads per second — the full rate of a disk provisioned for 40,000 operations per second** — at which point its latency rises to seventeen milliseconds and it begins to refuse three percent of requests rather than fall behind. It sustains **10,000 durable writes per second at a ninety-ninth-percentile latency of nine milliseconds**, each write flushed to disk before it is acknowledged. Under more load than it can finish, it refuses the excess and stays up; across every run it neither ran out of memory nor crashed.

I measured these numbers on data made to resemble OpenTelemetry signals — logs and trace spans — rather than on data chosen to flatter the database. The rows carry the kinds of attributes such signals carry: a few of high cardinality, such as the address of the client that originated a request, and many of low cardinality, such as the address of the backend that served it and the outcome of the call. Each row is about a kilobyte, and it compresses by about 2.8 to one, as telemetry of this kind does. The whole of it was sized at twice the memory available to the server, so that most reads reach the disk rather than being answered from memory. It is not yet heavily tuned, and I am already content with what one machine does.

## The benchmark in detail

Every figure above comes from one configuration, which was fixed throughout.

1. **The machine** was a single AWS `c7a.8xlarge` instance, an x86-64 server running Ubuntu 26.04.
2. **The disks** were two AWS io2 Block Express volumes, one for the data and one for the write-ahead log, each provisioned for 40,000 input-output operations a second.
3. **The memory** was held to 32 gibibytes by a control group, of which 16 gibibytes were the server's block cache.
4. **Compression** was Zstandard at level 6 throughout, and the read dataset was bulk-ingested already compressed at that level.
5. **The data** was shaped to resemble OpenTelemetry signals: each row carried a few attributes of high cardinality, such as the address of the client that began a request, and many of low cardinality, such as the address of the backend that served it and the outcome of the call. Each row was about a kilobyte, held roughly twenty-five cells, lived in a single column family, and compressed by about 2.8 to one to some 336 bytes on disk.
6. **The read dataset** was 409 million such rows, about 128 gibibytes on disk and twice the server's memory; after they were ingested, the operating system's page cache was dropped, so that nothing was left resident in memory from the loading and the reads had to reach the disk.
7. **The reads** were driven by ghz, the gRPC benchmarking tool, which drew its keys uniformly at random from the whole dataset so that no favoured part of it could settle in memory; because the dataset is larger than all the server's memory, the great majority of reads miss the cache and reach the disk, and the read rate is bounded by the 40,000 operations a second the disk provides.
8. **The writes** were durable single-row writes, each flushed to the log with `fsync` before it was acknowledged.
9. **Load shedding** was on by default, and was performed by the [Limen](https://rowkeydb.com/limen) library.

*The benchmarking automation that produced these numbers will be published shortly, to aid reproduction.*

### Reads

| Reads per second | p99 latency | Requests shed |
|---|---|---|
| 10,000 | 1 ms | 0% |
| 20,000 | 2 ms | 0% |
| 30,000 | 5 ms | 0% |
| 40,067 | 17 ms | 3% |

The server holds a ninety-ninth-percentile latency of five milliseconds at 30,000 reads a second while refusing nothing, and it serves 40,000 reads a second as the disk reaches its limit, at which point the latency rises and it begins to refuse a small fraction of the requests. An attempt to drive the rate higher still, to 50,000 reads a second, ended not in RowKeyDB faltering but in the load generator: ghz ran out of memory and was killed, while the database went on serving. The limit at this end of the table is the disk's, and then the client's; it was never the database's.

These rows each have one column family. A read of a row spread across several column families must gather them, and the figures for such rows will be different; co-located column families — locality groups — are the next piece of work, and are meant to bring rows of several families to the same performance for data of this kind and size.

### Durable writes

Every write measured here is durable. The server flushes it to the write-ahead log, with `fsync`, before it answers, so an acknowledged write survives an unannounced kill of the process.

| Durable writes per second | p99 latency | Requests shed |
|---|---|---|
| 2,000 | 3 ms | 0% |
| 5,000 | 3 ms | 0% |
| 8,000 | 5 ms | 0% |
| 9,000 | 7 ms | 0% |
| 10,000 | 9 ms | 0% |
| 11,000 | 128 ms | 0% |

The server sustains 10,000 durable writes a second at a ninety-ninth-percentile latency of nine milliseconds, and delivers about 11,000 at most, beyond which the latency rises steeply. The write rate is set by how quickly the log can be flushed to the disk, not by the number of operations the disk can perform, so it is a smaller figure than the read rate and a separate measurement. The way to raise it on one machine is a faster or a parallel write-ahead log, which is on the list of work below.

### Load shedding

RowKeyDB refuses work it cannot finish, early and cheaply, instead of accepting it and collapsing. This is done by [Limen](https://rowkeydb.com/limen), an open-source library, built in and on by default. When the read benchmark reached the disk's limit it shed three percent and kept answering the rest. Offered 14,000 writes a second, well past what the disk allows, it shed thirty-eight percent and stayed available. In neither case did it run out of memory, and in neither case did it crash.

## Download and verify

Binaries for x86-64 Linux are published under [Releases](https://github.com/rowkeydb-com/rowkeydb-releases/releases). This first beta is built for Ubuntu 24.04 and newer — that is, for any distribution carrying glibc 2.39 or later — and binaries that run on older systems will follow in due course. Every release is signed; before you run one, confirm it came from us:

```bash
gpg --locate-keys bgm@rowkeydb.com
gpg --fingerprint bgm@rowkeydb.com          # compare with the fingerprint below
gpg --verify SHA256SUMS.asc SHA256SUMS
sha256sum -c SHA256SUMS
```

The signing key fingerprint is:

```
A56D 0CCF 7F9E 66BD F5E9  7CAF 54BD 1DFD F41B D841
```

It is published here as [rowkeydb-signing-key.asc](rowkeydb-signing-key.asc), at [rowkeydb.com/security](https://rowkeydb.com/security), on the keyserver `keys.openpgp.org`, and on GitHub at `https://github.com/marete.gpg`. Check that the same forty characters appear in more than one of these before you trust a download. The source remains private during the beta.

## What comes next

Three pieces of work follow this release, in increasing order of size.

The first is co-located column families, or locality groups: storing several column families of a row together, so that reading such a row costs what reading a single-family row costs today.

The second is higher write throughput on a single machine. The write path is bounded by the speed of flushing the log to disk; a parallel or a faster write-ahead log will raise the rate.

The third, and the largest, is the multi-node version. Its design is written and has been reviewed. It will scale storage and performance in proportion to the machines given to it, with no separate coordination service to run, and it is meant to stand in for almost any Google Bigtable or HBase installation in practice.

## Run it, and tell me what breaks

Run RowKeyDB against your own workloads, and report what fails at [Issues](https://github.com/rowkeydb-com/rowkeydb-releases/issues). If you know someone still running HBase, please pass this along.
