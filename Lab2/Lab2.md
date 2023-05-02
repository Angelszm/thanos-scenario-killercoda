


## Deploy Prometheus Instance 
```
docker run -d --net=host --rm \
    -v root/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    -v root/prom-eu1:/prometheus \
    -u root \
    --name prometheus-0-eu1 \
    quay.io/prometheus/prometheus:v2.38.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9090 \
    --web.external-url=https://b2216f01-26e7-4dc0-87bd-c17a8c7ed56c-10-244-5-12-9090.papa.r.killercoda.com \
    --web.enable-lifecycle \
    --web.enable-admin-api

```

## Thanos Sidecar and Querier
```
docker run -d --net=host --rm \
    --name prometheus-0-eu1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.28.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --prometheus.url http://172.17.0.1:9090
```

```
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.28.0 \
    query \
    --http-address 0.0.0.0:9091 \
    --query.replica-label replica \
    --store 172.17.0.1:19190
```


## Step 2 - Object Storage Continuous Backup
Using Minio Engine and mostly keeps data in local disk 
```
mkdir /root/minio && \
docker run -d --rm --name minio \
     -v /root/minio:/data \
     -p 9000:9000 -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=melovethanos" \
     minio/minio:RELEASE.2019-01-31T00-31-19Z \
     server /data
```

### bucket_storage.yaml
```
type: S3
config:
  bucket: "thanos"
  endpoint: "172.17.0.1:9000"
  insecure: true
  signature_version2: true
  access_key: "minio"
  secret_key: "melovethanos"
```


## Let's run sidecar with Object Bucket Configuration : 
```
docker run -d --net=host --rm \
    -v $(pwd)/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    -v $(pwd)/prom-eu1:/prometheus \
    --name prometheus-0-eu1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.28.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --prometheus.url http://172.17.0.1:9090

```


## Step 3 - Fetching metrics from Bucket
- In this step, we will learn about Thanos Store Gateway and how to deploy it.
```
docker run -d --net=host --rm \
    -v /root/editor/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    --name store-gateway \
    quay.io/thanos/thanos:v0.28.0 \
    store \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191
```


### How to query Thanos store data?
```
docker stop querier && \
docker run -d --net=host --rm \
   --name querier \
   quay.io/thanos/thanos:v0.28.0 \
   query \
   --http-address 0.0.0.0:9091 \
   --query.replica-label replica \
   --store 172.17.0.1:19190 \
   --store 172.17.0.1:19191
```


## Step 4 - Thanos Compactor
- The Compactor is an essential component that operates on a single object storage bucket to compact, downsample, apply retention, to the TSDB blocks held inside, thus, making queries on historical data more efficient. It creates aggregates of old metrics (based upon the rules).
```
docker run -d --net=host --rm \
 -v /root/editor/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    --name thanos-compact \
    quay.io/thanos/thanos:v0.28.0 \
    compact \
    --wait --wait-interval 30s \
    --consistency-delay 0s \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19095
```

- The flag wait is used to make sure all compactions have been processed while --wait-interval is kept in 30s to perform all the compactions and downsampling very quickly. Also, this only works when when --wait flag is specified. Another flag --consistency-delay is basically used for buckets which are not consistent strongly. It is the minimum age of non-compacted blocks before they are being processed. Here, we kept the delay at 0s assuming the bucket is consistent.