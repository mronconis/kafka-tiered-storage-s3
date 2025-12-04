# Kafka tiered storage S3(MinIO)
The repository aims to show how Kafka tiered storage works using an S3 bucket on MinIO.

Kafka brokers always keep the latest data on the local storage tier. But the older data will be offloaded to the remote tier. The amount of data retained locally, the frequency of offloading, and other specifics depend on the tiered storage configuration (make sure to check out the current tiered storage [limitations](https://kafka.apache.org/documentation/#tiered_storage_limitation)).

## Setup
Create MinIO server:
```bash
oc apply -f minio.yaml
```

Create kafka cluster:
```bash
oc apply -f kafka.yaml
```

## Test
Produce test data:
```bash
oc apply -f job.yaml
```

## Verify
List the files in the directory where the local storage tier stores its data:
```bash
oc rsh demo-cluster-knp-broker-0 ls -l /var/lib/kafka/data/kafka-log0/tiered-storage-test-0
total 122864
...output omit...
-rw-r--r--. 1 1000790000 1000790000     5144 Dec  4 12:08 00000000000001857762.index
-rw-r--r--. 1 1000790000 1000790000 10482388 Dec  4 12:08 00000000000001857762.log
-rw-r--r--. 1 1000790000 1000790000      102 Dec  4 12:08 00000000000001857762.snapshot
-rw-r--r--. 1 1000790000 1000790000     1896 Dec  4 12:08 00000000000001857762.timeindex
...output omit...
```

Repeat the command just few minutes later and see that despite our topic having very long retention, the old segments are gone and new segments took their place:
```bash
oc rsh demo-cluster-knp-broker-0 ls -l /var/lib/kafka/data/kafka-log0/tiered-storage-test-0
total 12
-rw-r--r--. 1 1000790000 1000790000        0 Dec  4 12:09 00000000000002000000.log
-rw-r--r--. 1 1000790000 1000790000      102 Dec  4 12:09 00000000000002000000.snapshot
-rw-r--r--. 1 1000790000 1000790000 10485756 Dec  4 12:09 00000000000002000000.timeindex
-rw-r--r--. 1 1000790000 1000790000        8 Dec  4 11:58 leader-epoch-checkpoint
-rw-r--r--. 1 1000790000 1000790000       43 Dec  4 11:58 partition.metadata
```

So, where did the old segments go? They were offloaded to the remote storage tier:
```bash
mc alias set minio http://minio:9000 minioadmin minioadmin
mc ls -r minio/kafka-tiered-storage/
...output omit...
[2025-12-04 12:09:40 CET] 7.0KiB STANDARD tiered-storage-test-zjCAoo05TfOtzEaZ4BnsAw/0/00000000000001857762-qPXBijDMQ46h0ALQU_itqw.indexes
[2025-12-04 12:09:40 CET]  10MiB STANDARD tiered-storage-test-zjCAoo05TfOtzEaZ4BnsAw/0/00000000000001857762-qPXBijDMQ46h0ALQU_itqw.log
[2025-12-04 12:09:40 CET]   768B STANDARD tiered-storage-test-zjCAoo05TfOtzEaZ4BnsAw/0/00000000000001857762-qPXBijDMQ46h0ALQU_itqw.rsm-manifest
...output omit...
```
Note the file name `00000000000001857762-qPXBijDMQ46h0ALQU_itqw.log`, it is the remote copy of the local segment `00000000000001857762.log`.

## Troubleshooting
In the broker log you can see when he started copying segment `00000000000001857762.log` to remote storage and when it finished:
```log
2025-12-04 12:08:40 INFO [kafka-rlm-copy-thread-pool-2] RemoteLogManager$RLMCopyTask:1018 - [RemoteLogManager=0 partition=zjCAoo05TfOtzEaZ4BnsAw:tiered-storage-test-0] Copying 00000000000001857762.log to remote storage.
...
2025-12-04 12:08:40 INFO [kafka-rlm-copy-thread-pool-2] RemoteLogManager$RLMCopyTask:1091 - [RemoteLogManager=0 partition=zjCAoo05TfOtzEaZ4BnsAw:tiered-storage-test-0] Copied 00000000000001857762.log to remote storage with segment-id: RemoteLogSegmentId{topicIdPartition=zjCAoo05TfOtzEaZ4BnsAw:tiered-storage-test-0, id=qPXBijDMQ46h0ALQU_itqw}
```

## References
Special thanks to the [Strimzi project](https://strimzi.io/) Team!

- [Tiered Storage](https://strimzi.io/docs/operators/latest/deploying#tiered_storage)
- [Configure CR](https://strimzi.io/docs/operators/latest/deploying#ref-tiered-storage-str)
- [Strimzi Blog](https://strimzi.io/blog/2025/04/22/tha-various-tiers-of-apache-kafka-tiered-storage/)