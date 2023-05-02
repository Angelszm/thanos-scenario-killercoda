# thanos-scenario-killercoda
Lab1: https://killercoda.com/thanos/scenario/1-globalview



## Install Thanos Side Car 
docker run -d --net=host --rm \
    -v $(pwd)/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    --name prometheus-0-sidecar-eu1 \
    -u root \
    quay.io/thanos/thanos:v0.28.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url http://172.17.0.1:9090 && echo "Started sidecar for Prometheus 0 EU1"


### To check sidecar metrics: 
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: eu1
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['172.17.0.1:9090']
  - job_name: 'sidecar'
    static_configs:
      - targets: ['172.17.0.1:19090']



## Deploying Thanos Querier
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.28.0 \
    query \
    --http-address 0.0.0.0:29090 \
    --query.replica-label replica \
    --store 172.17.0.1:19190 \
    --store 172.17.0.1:19191 \
    --store 172.17.0.1:19192 && echo "Started Thanos Querier"
    
    
    
    ![image](https://user-images.githubusercontent.com/30867160/235598507-3e9aea2d-c198-4704-8610-a798a66340d8.png)
    
    
    Metrics 
    - prometheus_tsdb_head_series{cluster="eu1",instance="172.17.0.1:9090",job="prometheus"}
    - sum(prometheus_tsdb_head_series)
    - prometheus_tsdb_head_series

