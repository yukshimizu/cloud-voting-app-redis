# K8s monitoring with Elastic for Cloud Voting App

This repo includes examples of K8s monitoring with the Elastic Stack in different ways for  a multi-container application in a K8s cluster. The application interface has been built using Python/Flask. The data component is using Redis. The application is instrumented with Prometheus Python Client to demonstrate a custom metric `cloud_votes`.

## 1. Pre-Requisites
- Helm 3+
- K8s cluster (The codes in this repo are tested on AKS)
- Elastic Stack (7.13 or later)
    - Elasticsearch
    - Kibana
    - APM
    
The easiest way to run the examples is spin up a managed K8s cluster and an Elastic Stack cluster on Elastic Cloud. The following procedure are assumed using AKS and Elastic Cloud.

## 2. Using CNCF projects
This example uses Prometheus for metrics, Fluentd for logging, Elasticsearch for datastore, and Kibana for visualization.

### 2.1 Create container images
Use the sample `docker-compose-cncf.yaml` file to create the container image, download the Redis image, and start the application.
```buildoutcfg
docker-compose -f docker-compose-cncf.yaml up -d
```
To see the running application, enter http://localhost:8080 in a local web browser. After testing, the running containers can be stopped and removed.
```buildoutcfg
docker-compose -f docker-compose-cncf.yaml down
```
Next, tag the container image assuming you already have a container registry (change the following container_registry_name).
```buildoutcfg
docker tag cloud-vote-front <container_registry_name>/cloud-vote-front:v1
```
Then, push images to registry with appropriate way for your registry.
```buildoutcfg
docker push <container_regsistry_name>/cloud-vote-front:v1
```

### 2.2 Deploy Fluentd
Fluentd ingests logs to `logstash-*` indices by default. Before running Fluentd, it's better to create an index template for `logstash-*` indices.
```buildoutcfg
PUT _template/logstash
{
  "index_patterns": [
    "logstash-*"
  ],
  "mappings": {
    "dynamic_templates": [
      {
        "message_field": {
          "path_match": "message",
          "mapping": {
            "norms": false,
            "type": "text"
          },
          "match_mapping_type": "string"
        }
      },
      {
        "another_message_field": {
          "path_match": "MESSAGE",
          "mapping": {
            "norms": false,
            "type": "text"
          },
          "match_mapping_type": "string"
        }
      },
      {
        "strings_as_keyword": {
          "mapping": {
            "ignore_above": 1024,
            "type": "keyword"
          },
          "match_mapping_type": "string"
        }
      }
    ],
    "properties": {
      "@timestamp": {
        "type": "date"
      }
    }
  }
}
```
Assuming your K8s cluster is running already, create a ConfigMap. Please refer to https://medium.com/kubernetes-tutorials/cluster-level-logging-in-kubernetes-with-fluentd-e59aa2b6093a for more details.
```buildoutcfg
kubectl create configmap fluentd-conf --from-file=cncf-projects/kubernetes.conf --namespace=kube-system
```
Once creating the ConfigMap, deploy Fluentd as a DaemonSet. Don't forget to change the following environment variables related to Elasticsearch cluster in `cncf-projects/fluentd-daemonset-elasticsearch-rbac.yaml`.
```buildoutcfg
# Change the followings in cncf-projects/fluentd-daemonset-elasticsearch-rbac.yaml.
- name:  FLUENT_ELASTICSEARCH_HOST
  value: "elasticsearch-logging"
- name:  FLUENT_ELASTICSEARCH_PORT
  value: "9200"
- name: FLUENT_ELASTICSEARCH_SCHEME
  value: "http"
- name: FLUENT_ELASTICSEARCH_USER
  value: "elastic"
- name: FLUENT_ELASTICSEARCH_PASSWORD
  value: "changeme"
  
# Deploy Fluentd
kubectl create -f cncf-projects/fluentd-daemonset-elasticsearch-rbac.yaml
```
### 2.3 Deploy Metricbeat
Before deploying Prometheus, we need to deploy Metricbeat as a Deployment so that we can ingest metrics from Prometheus to Elasticsearch. In this case, we use Prometheus module of Metricbeat https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-prometheus-remote_write.html.
Don't forget to change the following environment variables related to Elasticsearch cluster in `cncf-projects/metricbeat-kubernetes-deployment.yaml`.
```buildoutcfg
# Change the followings in cncf-projects/metricbeat-kubernetes-deployment.yaml
env:
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
- name: ELASTIC_CLOUD_ID
  value:
- name: ELASTIC_CLOUD_AUTH
  value:

# Deploy Metricbeat
kubectl create -f cncf-projects/metricbeat-kubernetes-deployment.yaml
```
### 2.4 Deploy Prometheus
We can use Helm charts for deploying Prometheus here. In order to customize default behavior including enable remote_write, `cncf-projects/prometheus_custom.yaml` is used. Please refer to https://github.com/prometheus-community/helm-charts for more details.
```buildoutcfg
helm install prometheus -n monitoring --create-namespace prometheus-community/prometheus -f cncf-projects/prometheus_custom.yaml
```

### 2.5 Deploy App
Finally, deploy Cloud Voting App. You need to change the image name in `cncf-projects/cloud-vote-all-in-one-redis-aks-prometheus.yaml` to the name pushed to your registry.
```buildoutcfg
# Change image name in cncf-projects/cloud-vote-all-in-one-redis-aks-prometheus.yaml
...
containers:
- name: cloud-vote-front
  image: your image name
...

# Deploy App
kubectl create -f cncf-projects/cloud-vote-all-in-one-redis-aks-prometheus.yaml
```
Once App is running, you can get external IP with following command and access to the service via browser.
```buildoutcfg
kubectl get service cloud-vote-front
```
### 2.6 Access to Kibana
Now, you can access to metrics and logs via Kibana. Find out metricbeat-* indices for metrics from Prometheus and logstash-* indices for logs from Fluentd.

## 3. Elastic way
This example fully utilizes the Elastic Stack capabilities, which includes Elasticsearch, Kibana, Beats and APM. 

### 3.1 Create container images
Use the sample `docker-compose-elastic.yaml` file to create the container image, download the Redis image, and start the application. Because the app is instrumented with Elastic APM agent for collecting Prometheus custom metrics, you need to change APM related variables in `docker-compose-elastic.yaml`. The --build argument is used to instruct Docker Compose to re-create the application image.
```buildoutcfg
# Change SECRET_TOKEN and SERVER_URL in docker-compose-elastic.yaml
cloud-vote-front:
  build: ./cloud-vote-elastic
  image: cloud-vote-front
  container_name: cloud-vote-front
  environment:
    REDIS: cloud-vote-back
    SERVICE_NAME: cloud-voting
    SECRET_TOKEN: APM Server secret token
    SERVER_URL: APM Server URL
    ENVIRONMENT: Development
    PROMETHEUS_METRICS: "True"
...
# Run app
docker-compose -f docker-compose-elastic.yaml up --build -d
```
To see the running application, enter http://localhost:8080 in a local web browser. After testing, the running containers can be stopped and removed.
```buildoutcfg
docker-compose -f docker-compose-elastic.yaml down
```
Next, tag the container image assuming you already have a container registry (change the following container_registry_name).
```buildoutcfg
docker tag cloud-vote-front <container_registry_name>/cloud-vote-front:v2
```
Then, push images to registry with appropriate way for your registry.
```buildoutcfg
docker push <container_regsistry_name>/cloud-vote-front:v2
```

### 3.2 Deploy Filebeat
We use Filebeat for collecting all the logs as against the previous example which uses Fluentd. Filebeat is deployed as a DaemonSet. Please refer to https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html for more details. Don't forget to change the following environment variables related to Elasticsearch cluster in `elastic/filebeat-kubernetes.yaml`.
```buildoutcfg
# Change the followings in elastic/filebeat-kubernetes.yaml
env:
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
- name: ELASTIC_CLOUD_ID
  value:
- name: ELASTIC_CLOUD_AUTH
  value:

# Deploy Metricbeat
kubectl create -f elastic/filebeat-kubernetes.yaml
```
### 3.3 Deploy Metricbeat
We use Metricbeat for collecting all the metrics as against the previous example which uses Prometheus. Before deploying Metricbeat, it's better to deploy `kube-state-metrics`. Please refer to https://github.com/kubernetes/kube-state-metrics for more details.
```buildoutcfg
kubectl apply -f examples/standard
```
And, Metricbeat is also deployed as a DaemonSet. Please refer to https://www.elastic.co/guide/en/beats/metricbeat/current/running-on-kubernetes.html for more details. Don't forget to change the following environment variables related to Elasticsearch cluster in `elastic/metricbeat-kubernetes.yaml`.
```buildoutcfg
# Change the followings in elastic/metricbeat-kubernetes.yaml
env:
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
- name: ELASTIC_CLOUD_ID
  value:
- name: ELASTIC_CLOUD_AUTH
  value:

# Deploy Metricbeat
kubectl create -f elastic/metricbeat-kubernetes.yaml
```

### 3.4 Deploy App
Finally, deploy Cloud Voting App. You need to change the image name in `elastic/cloud-vote-all-in-one-redis-aks.yaml` to the name pushed to your registry. Also, you need to change Elastic APM related variables such as SECRET_TOKEN and SERVER_URL for APM instrumentation.
```buildoutcfg
# Change image name in elastic/cloud-vote-all-in-one-redis-aks.yaml
...
containers:
- name: cloud-vote-front
  image: your image name
...
  env:
  - name: REDIS
    value: "cloud-vote-back"
  - name: SERVICE_NAME
    value: "cloud-voting"
  - name: SECRET_TOKEN
    value: "APM Server secret token"
  - name: SERVER_URL
    value: "APM Server URL"
  - name: ENVIRONMENT
    value: "Production"
  - name: PROMETHEUS_METRICS
    value: "True"
...
# Deploy App
kubectl create -f elastic/cloud-vote-all-in-one-redis-aks.yaml
```
Once App is running, you can get external IP with following command and access to the service via browser.
```buildoutcfg
kubectl get service cloud-vote-front
```
### 3.5 Access to Kibana
Now, you can access to metrics and logs via Kibana. Find out metricbeat-* indices for metrics and filebeat-* indices for logs. You can find out apm-* indices for custom metrics of Prometheus in addition to usual APM data. And, you can also fully utilize Elastic Observability App in Kibana.