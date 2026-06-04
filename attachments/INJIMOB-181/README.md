# OBSRV K8S Cluster Setup using RKE

## Overview
This is a guide to set up Kubernetes cluster on Virtual Machines using [RKE](https://www.rancher.com/products/rke). This cluster would host all OBSRV modules.

## Introduction
Obsrv is a platform that enables you to manage your data workflows from ingestion, all the way to reporting. With a high level of abstraction, anyone can create datasets, set up connectors to various data sources, define data manipulations and export the aggregations via multiple data visualization tools, all with minimal technical debt/knowledge. Obsrv is built under the hood using the latest open source tools that can be swapped, plugged in or out depending on use cases. Obsrv comes with a set of microservices, APIs, and some utility SDKs. Obsrv also has a built-in open data cataloging and publishing capability.

## Hardware
Obsrv can support a volume of 5 million events per day with an average size of each event to be around 5 kb with the following specifications.
* Kubernetes version of 1.25 or greater
* Minimum of 16 cores of CPU
* Minimum of 64 GB of RAM
* PersistentVolume support in the Kubernetes cluster

## Getting Started with Obsrv Deployment Using Helm
Sunbird Obsrv is a high-performance, cost-effective data stack with several components such as ingestion, querying, processing, backup, visualisation and monitoring. Obsrv 2.0 can be either installed using Helm (Kubernets Package Manager).

## Prerequisites
Obsrv runs completely on a Kubernetes cluster. A completely functional Kubernetes cluster is expected for a seamless Obsrv installation.

* Ansible
* Docker
* rke (rke version: v1.3.10)
* istioctl (istioctl version: v1.15.0)

### Helm
Make sure Helm should be Install in your system.
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Virtual machines Setup
* Set up passwordless SSH.
```bash
git clone https://github.com/mosip/k8s-infra.git
cd k8s-infra
git checkout develop
cd k8s-infra/mosip/on-prem
```
* Create copy of `hosts.ini.sample` as `hosts.ini` Update the IP addresses.
## Ports
*  Open ports on each of the nodes.
  ```bash
  ansible-playbook -i hosts.ini ports.yaml
```
## Docker
* Install docker on all nodes.
```bash
ansible-playbook -i hosts.ini docker.yaml
```

## RKE cluster setup
* Make sure [Rancher Cluster](https://github.com/mosip/k8s-infra/tree/main/rancher) should be installed.
* Create a cluster config file.
```bash
rke config
```
- controlplane, etcd, worker: Specify controlplane, etc on at least three nodes. All nodes may be worker.
- Use default canal networking model
- Keep the Pod Security Policies disabled.
- Sample configuration options:
```bash
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
[+] Number of Hosts [1]:
[+] SSH Address of host (1) [none]: <node1-ip>
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (<node1-ip>) [none]:
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (<node1-ip>) [none]:
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (<node1-ip>) [ubuntu]:
[+] Is host (<node1-ip>) a Control Plane host (y/n)? [y]: y
[+] Is host (<node1-ip>) a Worker host (y/n)? [n]: y
[+] Is host (<node1-ip>) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (<node1-ip>) [none]: node2
[+] Internal IP of host (<node1-ip>) [none]:
[+] Docker socket path on host (<node1-ip>) [/var/run/docker.sock]:
[+] Network Plugin Type (flannel, calico, weave, canal) [canal]:
[+] Authentication Strategy [x509]:
[+] Authorization Mode (rbac, none) [rbac]:
[+] Kubernetes Docker image [rancher/hyperkube:v1.17.17-rancher1]:
[+] Cluster domain [cluster.local]:
[+] Service Cluster IP Range [10.43.0.0/16]:
[+] Enable PodSecurityPolicy [n]:
[+] Cluster Network CIDR [10.42.0.0/16]:
[+] Cluster DNS Service IP [10.43.0.10]:
[+] Add addon manifest URLs or YAML files [no]:
```
* While opting for roles for different nodes follow below points:

  - In case of odd no of total nodes of cluster opt for (n+1/2) nodes with Control plane, etcd host and worker host role and rest of the nodes with Worker host and etcd host role.
  - In case of even no of total nodes of cluster opt for (n/2) nodes with Control Plane, etcd host and worker host role and rest of the node with Worker host and etcd host role.
* Remove the default Ingress install by editing `cluster.yaml`:
```bash
ingress:
  provider: none
  ```
* Add the name of the kubernetes cluster in `cluster.yml`:
```bash
cluster_name: obsrv-cluster_name
```
* [Sample config file](https://github.com/mosip/k8s-infra/blob/main/mosip/on-prem/sample.cluster.yml) is provider in this folder.
* Bring up the cluster:
```bash
 rke up
```
* After successful creation of cluster a `kube_config_cluster.yaml` will get created. Copy the file to `$HOME/.kube` folder.
```bash
cp kube_config_cluster.yml $HOME/.kube/<cluster_name>_config
chmod 400 $HOME/.kube/<cluster_name>_config
```
* To set this file as global default for `kubectl`, make sure you have a copy of existing `$HOME/.kube/config`.
```bash
cp  $HOME/.kube/<cluster_name>_config  $HOME/.kube/config
```
* Alternatively, set `KUBECOFIG` env variable:
```bash
export KUBECONFIG=$HOME/.kube/<cluster_name>_config
```
* Test
```bash
kubectl get nodes
```
## Register the cluster with Rancher
* Login as admin in Rancher console
* Select `Import Existing1 for cluster addition.
* Select the `Generic` as cluster type to add.
* Fill the `Cluster Name` field with unique cluster name and select `Create`.
* You will get the kubecl commands to be executed in the kubernetes cluster

```bash
eg.
kubectl apply -f https://rancher.e2e.mosip.net/v3/import/pdmkx6b4xxtpcd699gzwdtt5bckwf4ctdgr7xkmmtwg8dfjk4hmbpk_c-m-db8kcj4r.yaml
```
* Wait for few seconds after executing the command for the cluster to get verified.
* Your cluster is now added to the rancher management server.

### Helm Dependencies
* Run the following helm repo add command to download the required dependencies for running Obsrv.
  - monitoring -
  - redis -
  - loki (version - 4.8.0 ) -
  - promtail (version - 6.9.3) -
  - velero (version - 3.1.6 ) -
```bash
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo update
```
```bash
helm repo add https://charts.bitnami.com/bitnami
helm repo update
```

```bash
helm repo add https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm repo add https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm repo add https://vmware-tanzu.github.io/helm-charts
helm repo update
```

## Deployement Stpes for persistent storage and object storage (s3)
### 1. Longhorn
* Install [Longhorn](https://github.com/mosip/k8s-infra/blob/main/longhorn/README.md) for persistent storage. OR you can Install Longhorn from RancherUI also.

### 2. MinIO
* Install [MinIO](https://github.com/mosip/mosip-infra/tree/master/deployment/v3/external/object-store/minio) for high performance, distributed object storage system.
  - Log into the minio with credential and create following buckets.

* The following list of buckets/containers need to be created for different services to store the data. This is applicable to Object Storage such as MinIO/Ceph as well.
  - flink-checkpoints
  - velero-backup
  - obsrv

### 3. Istio
* Make sure [Istio](https://github.com/mosip/k8s-infra/blob/main/mosip/on-prem/istio/README.md) should be installed
## Sorce Code
* Clone the [obsrv-automation](https://github.com/Sunbird-Obsrv/obsrv-automation) github repository. The required list of helm charts to deploy Obsrv will be under the terraform/modules/helm directory.
```bash
git clone https://github.com/Sunbird-Obsrv/obsrv-automation.git
cd obsrv-automation/terraform/modules/helm
```
## Deployment Instructions
Helm package manager provides an easy way to install specific components using a generic command. Configurations can be overriden by updating the values.yaml file in the respective Helm charts.
```bash
Example
helm upgrade --install --atomic <release_name> <chart_name> -n <namespace> -f <path/values.yaml> --create-namespace --debug
```
### Postgres
Postgres is a RDBMS database which is used as the metadata store
```bash
helm upgrade --install --atomic obsrv-redis redis/redis -n redis -f redis/values.yaml --create-namespace --debug
```
### Redis
Redis is an in-memory key-value store primarily used as a distributed cache
```bash
helm upgrade --install --atomic obsrv-redis redis/redis -n redis -f redis/values.yaml --create-namespace --debug
```
### Prometheus
Prometheus is a monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
```bash
helm upgrade --install --atomic monitoring monitoring/kube-prometheus-stack -n monitoring -f monitoring/values.yaml --create-namespace --debug
```
### Kafka
Apache Kafka is a distributed event store and stream-processing platform.
```bash
helm upgrade --install --atomic kafka kafka/kafka-helm-chart -n kafka --create-namespace --debug
```
The following list of kafka topics are created by default. If you would like to add more topics to the list, you can do so by adding it to provisioning.topics configuration in the [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/kafka/kafka-helm-chart/values.yaml) file.
* dev.ingest
* masterdata.ingest

### Druid
Druid is a high performance, real-time analytics database that delivers sub-second queries on streaming and batch data at scale
#### Druid CRD
```bash
helm upgrade --install --atomic druid-operator druid_operator/druid-operator-helm-chart -n druid-raw --create-namespace --debug
```

#### Druid Cluster
* Druid requires the following set of configurations to be provided for specific storage systems such as AWS S3, MinIO

  - Create `Accress Key` and `Secret Key` from the MinIO
  - Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/druid_raw_cluster/druid-raw-cluster-helm-chart/values.yaml) file and update as per your persistence storage, MinIO access and secret keys etc.
  - replace following placeholders:
```bash
     storageClass: "gp2" as storageClass: "longhorn"`
     druid_metadata_storage_connector_password: "druidraw123"` as `druid_metadata_storage_connector_password: "druid_raw"`
   ```

```bash
# AWS S3 Details
# Use the ClusterIP of the MinIO service instead of the Kubernetes service name in the endpoint_url
s3_access_key: "KP335MO3XT1A1Z7XPSWI"
s3_secret_key: "r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
s3_bucket: "obsrv"
druid_s3_endpoint_url: "http://10.43.209.124:9000/" 
druid_s3_endpoint_signingRegion: "us-east-2"
```
* Add `druid_indexer_logs_s3Bucket:"obsrv"
  `, `druid_indexer_logs_directory:"backups/druid/druid-task-logs"`,  lines as given below in values.yaml file.

* Replace `druid_broker_max_heap_size: 512M` as `druid_broker_max_heap_size: 1024M
  ` and `druid_broker_pod_memory_limit: 750Mi` as `druid_broker_pod_memory_limit: 1300Mi` in values.yaml file.
```bash
# Indexing service logs
# For local disk (only viable in a cluster_type if this is a network mount):
druid_indexer_logs_type: "s3"
druid_indexer_logs_s3Bucket: "obsrv"
druid_indexer_logs_container: ""
druid_indexer_logs_prefix: "backups/druid/druid-task-logs"
druid_indexer_logs_directory: "backups/druid/druid-task-logs"
```

* Edit [druid_statefulset.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/druid_raw_cluster/druid-raw-cluster-helm-chart/templates/druid_statefulset.yaml) file as well.
  - Add `druid.indexer.logs.s3Bucket={{ .Values.druid_indexer_logs_s3Bucket }}`, `    druid.indexer.logs.s3Prefix={{ .Values.druid_indexer_logs_prefix }}` and `	    druid.indexer.logs.directory={{ .Values.druid_indexer_logs_directory }}` in druid_statefulset.yaml file
```bash
    # # Indexing service logs
    # # For local disk (only viable in a cluster if this is a network mount):
    druid.indexer.logs.type={{ .Values.druid_indexer_logs_type }}
    druid.indexer.logs.container={{ .Values.druid_indexer_logs_container }}
    druid.indexer.logs.prefix={{ .Values.druid_indexer_logs_prefix }}
    druid.indexer.logs.s3Bucket={{ .Values.druid_indexer_logs_s3Bucket }}
    druid.indexer.logs.s3Prefix={{ .Values.druid_indexer_logs_prefix }}
    druid.indexer.logs.directory={{ .Values.druid_indexer_logs_directory }}
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic druid-raw druid_raw_cluster/druid-raw-cluster-helm-chart -n druid-raw --create-namespace --debug
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  creationTimestamp: "2023-08-16T10:30:59Z"
  generation: 6
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        .: {}
        f:selector:
          .: {}
          f:istio: {}
        f:servers: {}
    manager: agent
    operation: Update
    time: "2023-08-16T10:30:59Z"
  name: druid
  namespace: druid-raw
  resourceVersion: "9587390"
  uid: b9e5fb62-9e75-450f-a7f7-ee0b7934c9e7
spec:
  selector:
    istio: ingressgateway-internal
  servers:
  - hosts:
    - druid.obsrv.mosip.net
    port:
      name: http
      number: 80
      protocol: HTTP
 ```
- VirtualServices

```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/3SRwY6cMAyG38XHCthhBgrkNSrtZTUHk3iWdCBJY7PTFcq7V0mnVS894R99+WwnB2CwrxTZegcKHMnDx7t1741lsb6x/uWjnUmwhQru1hlQ8Gqj7Lh+o/hhNUEFGwkaFAR1ADrnBcV6xzn6+TtpYZImWt9oFFkpS20WdXSj0zR19Xk6TXXXnrAe2xvWA7Wt7vtp7OYWUgUrzrQWHYbQ3PeZoiMhziLtt+AdOQEFm2cboPpv0wV5AQVT21One+z6aTrRhbqzHsbLcBmM7uZ2noYznb/SqHNrhxuBAhN3a+B35ID677864iNzHEjnAd9R6IGfDOoNyg3W/MlC24t1QtHhCtcKFs9SiC8liQRQbwcshIZi2TPSj51YcslUPj/rm48PjIZMHaIXD6qcZEgpVbCh6KVYntUBwUcBNY7jmNK1guh3oUIYYrGuvFHm8jT/rlMXMjJUT8UBbt9min9k6ZquKf0KAAD//0oZDvg6AgAA
    objectset.rio.cattle.io/id: 4efe0994-2909-410a-81fa-7e11c55984b1
  creationTimestamp: "2023-08-16T10:08:59Z"
  generation: 9
  labels:
    app.kubernetes.io/component: mosip
    objectset.rio.cattle.io/hash: 915e4c5a45990e3e42c783737dc4b1b972e26e8c
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:objectset.rio.cattle.io/applied: {}
          f:objectset.rio.cattle.io/id: {}
        f:labels:
          .: {}
          f:app.kubernetes.io/component: {}
          f:objectset.rio.cattle.io/hash: {}
      f:spec:
        .: {}
        f:gateways: {}
        f:hosts: {}
        f:http: {}
    manager: agent
    operation: Update
    time: "2023-08-16T10:08:59Z"
  name: druid
  namespace: druid-raw
  resourceVersion: "9587465"
  uid: 1e7ac965-7023-44b6-94cc-de84e9a1e304
spec:
  gateways:
  - druid
  hosts:
  - '*'
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: druid-raw-routers
        port:
          number: 8888
```
## Dataset-API
* This service provides metadata APIs related to various resources such as datasets/datasources in Obsrv. The following configurations need to be specified in the [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/dataset_api/dataset-api-helm-chart/values.yaml) file.
  - Edit values.yaml file and update as per requirement
  - replace following placeholders:
```bash
  CONTAINER: "obsrv-sb-dev-165052186109" --> CONTAINER: "obsrv"
  ```
* Edit [deployment.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/dataset_api/dataset-api-helm-chart/templates/deployment.yaml) file as well.
  - Replace `type: "LoadBalancer"` as `type: ClusterIP`
  - Add `port` and `targetPort` as given below.
```bash
	spec:
  type: ClusterIP
  ports:
    - name: http-{{ .Chart.Name }}
      protocol: TCP
      port: {{ .Values.network.port }}
      targetPort: {{ .Values.network.targetport }}
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic dataset-api dataset_api/dataset-api-helm-chart -n dataset-api --create-namespace --debug 
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  annotations:
    meta.helm.sh/release-namespace: dataset-api
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/3SQsY7bMAyG34Wz7bMTO7a1dujaoehS3EBJvFg9WRRENoci8LsXSrsdslEQ+P0f/ztgDj+oSOAEBhLpB5f3kK5dEA3cBX65DZYUB2jgPSQPBr6i0gf+gQZ2UvSoCOYOmBIrauAk9Vm/uo3i3sn2UigSCrUJd5KMjsBA3RPSFnOABtj+IqdC2pXAnUPVSDU81MAJp/FsL0Prer+2I63Yol98Oy7jyb6dFhxWC0cDES3FR/oz3IaygYGL7U84u8VPdhiwH72f5rd1mU6rX86zu5ynyzjPPVZodf6k++yQowHJ5KqCUCSnXOr86BIMhHQtJHL9V2AbklJJGB9rVG5UBMzPO2wsWidgK+XW7Swhd4kUXhvIXLQi/2tlFq3M6vR7t1TADOe+7xvIhZUdRzDw/cs3OI7X4/gbAAD//6u9v3buAQAA
    objectset.rio.cattle.io/id: 5a543b61-c0d9-4e9a-ad8d-4842bf28a19b
  creationTimestamp: "2023-08-04T14:04:21Z"
  generation: 9
  labels:
    objectset.rio.cattle.io/hash: 6b02a7c8d5b11a04dd57f98529d837c63564770a
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-namespace: {}
          f:objectset.rio.cattle.io/applied: {}
          f:objectset.rio.cattle.io/id: {}
        f:labels:
          .: {}
          f:objectset.rio.cattle.io/hash: {}
      f:spec:
        .: {}
        f:selector:
          .: {}
          f:istio: {}
        f:servers: {}
    manager: agent
    operation: Update
    time: "2023-08-04T14:04:21Z"
  name: dataset-api
  namespace: dataset-api
  resourceVersion: "15261894"
  uid: 1ce2abf1-7d00-48f5-872e-25a00f248c62
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - dataset-api.obsrv.mosip.net
    port:
      name: dataset-api
      number: 80
      protocol: HTTP
```
* VirtualServices
```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    meta.helm.sh/release-namespace: dataset-api
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/3SPzY7cIBCE36WPEfbC+CcOrxFpLysfGtwek7XBgp4ZRRbvHuFJlMNqbjTdVfXVAbi7d4rJBQ8aPPEjxE/nr7VL7ELtwttdGWJUIODT+Qk0vLvIN1x/Urw7SyBgI8YJGUEfgN4HRnbBpzKWVb3QutVpeYu0EiaqPG6UdrQEGoouEVe4OxAQzC+ynIjr6EJtkXmlwuBKrlLz0CLOlcQBq5ZsV5nux1A1RM1llnPbdxfIAq7kKZ4IoJWAFQ2tJ8wr9wXTAhqG3ijb96ZpOsLBfpfdYLG/9KQG1UyWrJlnibItGaXCF/pXvbKAtJMtCFdkeuDvBPqjnPy7GAUsIfH5/a1MbHfQHwdsyHY5X3uIDFo1Uso8CojhxnQuJkrs/N++x+nzzP8P9tQe4G+boQj6NMl5zGPOfwIAAP//53/ytQICAAA
    objectset.rio.cattle.io/id: 11f84aaf-0a8a-4ec5-b598-3ee32f0f4652
  creationTimestamp: "2023-08-04T14:05:55Z"
  generation: 14
  labels:
    objectset.rio.cattle.io/hash: 86b1c66b335ea8c7058ca626e1813dcecbff0a04
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-namespace: {}
          f:objectset.rio.cattle.io/applied: {}
          f:objectset.rio.cattle.io/id: {}
        f:labels:
          .: {}
          f:objectset.rio.cattle.io/hash: {}
      f:spec:
        .: {}
        f:gateways: {}
        f:hosts: {}
        f:http: {}
    manager: agent
    operation: Update
    time: "2023-08-16T14:17:36Z"
  name: dataset-api
  namespace: dataset-api
  resourceVersion: "14865136"
  uid: 31f346c6-5cc7-4112-8692-3b58786f782c
spec:
  gateways:
  - dataset-api
  hosts:
  - '*'
  http:
  - corsPolicy:
      allowCredentials: true
      allowHeaders:
      - Accept
      - Accept-Encoding
      - Accept-Language
      - Connection
      - Content-Type
      - Cookie
      - Host
      - Referer
      - Sec-Fetch-Dest
      - Sec-Fetch-Mode
      - Sec-Fetch-Site
      - Sec-Fetch-User
      - Origin
      - Upgrade-Insecure-Requests
      - User-Agent
      - sec-ch-ua
      - sec-ch-ua-mobile
      - sec-ch-ua-platform
      - x-xsrf-token
      - xsrf-token
      allowMethods:
      - GET
      - POST
      - PATCH
      - PUT
      - DELETE
      allowOrigins:
      - prefix: https://dataset-api.obsrv.mosip.net
    headers:
      request:
        set:
          x-forwarded-proto: https
    match:
    - uri:
        prefix: /
    route:
    - destination:
        host: dataset-api-service
        port:
          number: 3000
```
## Flink Streaming Jobs
Flink jobs are used to process and enrich the data ingested into Obsrv in near-realtime.
### Configuration Overrides

- Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/flink/flink-helm-chart/values.yaml) as per requirement
```bash
# AWS S3 Details
s3_access_key: "admin"
s3_secret_key: "sDABZCoVOK"
s3_endpoint: "http://10.43.209.124:9000"
s3_path_style_access: ""

# Under base_config in the values.yaml
base.url: s3://flink-checkpoints

# Under redis and redis-meta in the values.yaml
host = redis-master.redis.svc.cluster.local
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic merged-pipeline flink/flink-helm-chart -n flink --set image.registry=sunbird --set image.repository=sb-obsrv-merged-pipeline --create-namespace --debug
```
## Secor
* Secor backups are performed from various kafka topics which are part of the data processing pipeline. The following list of backup names need to be replaced in the below mentioned command.
  - Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/secor/secor-helm-chart/values.yaml) file as per requirement
  - Add below mention preoperties in values.yaml file
```bash
aws_access_key: "KP335MO3XT1A1Z7XPSWI"
aws_secret_key: "r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
aws_region: "us-east-2"
aws_endpoint: "http://10.43.209.124:9000/"
secor_s3_bucket: "secor"
```
- Replace `cloud_storage_bucket: "obsrv-sb-dev-165052186109"` as `cloud_storage_bucket: "secor"` and `storageClass: "gp2"` as `storageClass: "longhorn"`
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
* List of backup names
* ingest-backup
* extractor-duplicate-backup
* extractor-failed-backup
* raw-backup
* failed-backup
* invalid-backup
* unique-backup
* duplicate-backup
* denorm-backup
* denorm-failed-backup
* system-stats
* system-events

###### NOTE: Deploy secor with above bucket name, add bucket name in the below command
```bash
helm upgrade --install --atomic <backup_name> secor/secor-helm-chart -n secor --create-namespace
```
## Velero

* Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/velero/values.yaml) file as given below

```bash
	  secretContents:
    cloud: |
      [default]
      aws_access_key_id="KP335MO3XT1A1Z7XPSWI"
      aws_secret_access_key="r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
configuration:
  provider: "aws"
  backupStorageLocation:
    bucket: "velero"
    config:
      region: ""
  volumeSnapshotLocation:
    name: default
    config:
      region: ""
 ```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic velero velero/velero -n velero -f velero/values.yaml --create-namespace --debug --version 3.1.6
```

## Monitoring Services
#### Monitoring Dashboards
```bash
helm upgrade --install --atomic grafana-configs grafana_configs/grafana-configs-helm-chart -n monitoring --create-namespace --debug
```
#### Monitoring Alert Rules
```bash
helm upgrade --install --atomic alertrules alert_rules/alert-rules-helm-chart -n monitoring --create-namespace --debug
```
#### Druid Exporter
```bash
helm upgrade --install --atomic druid-exporter druid_exporter/druid-exporter-helm-chart -n druid-raw --create-namespace --debug
```
#### Kafka Exporter
```bash
helm upgrade --install --atomic kafka-exporter kafka_exporter/kafka-exporter-helm-chart -n kafka --create-namespace --debug
```

#### Postgres Exporter
* Replace Service type `type: NodePort` as `type: ClusterIP`
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
````
```bash
helm upgrade --install --atomic postgresql-exporter postgresql_exporter/postgresql-exporter-helm-chart -n postgresql --create-namespace --debug
```
#### Loki
```bash
helm upgrade --install --atomic loki loki/loki -n loki -f loki/values.yaml --create-namespace --debug --version 4.8.0
```
### Ingestion
* This helm chart is used to submit the default ingestion tasks required for the system statistics events
```bash
helm upgrade --install --atomic submit-ingestion submit_ingestion/submit-ingestion-helm-chart -n submit-ingestion --create-namespace --debug 
```
## Visualization
#### Superset
* Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/superset/superset-helm-chart/values.yaml) file as per requirement
* Replace redis-host `	redis_host: obsrv-redis-master.redis.svc.cluster.local` as `	redis_host: redis-master.redis.svc.cluster.local` and service `
  type: LoadBalancer` as `
  type: ClusterIP`

* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
````
```bash
helm upgrade --install --atomic superset superset/superset-helm-chart -n superset --create-namespace --debug
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  creationTimestamp: "2023-08-16T14:04:04Z"
  generation: 1
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        .: {}
        f:selector:
          .: {}
          f:istio: {}
        f:servers: {}
    manager: agent
    operation: Update
    time: "2023-08-16T14:04:04Z"
  name: superset
  namespace: superset
  resourceVersion: "9632248"
  uid: 0bdead67-4291-41d3-a78d-d9f79328a916
spec:
  selector:
    istio: ingressgateway-internal
  servers:
  - hosts:
    - superset.obsrv.mosip.net
    port:
      name: http
      number: 80
      protocol: HTTP
```
* VirtualServices
```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    objectset.rio.cattle.io/applied: H4sIAAAAAAAA/3SRwY6cMAyG38XHCthhBgrkNSrtZTUHk3iWdCBJY7PTFcq7V0mnVS894R99+WwnB2CwrxTZegcKHMnDx7t1741lsb6x/uWjnUmwhQru1hlQ8Gqj7Lh+o/hhNUEFGwkaFAR1ADrnBcV6xzn6+TtpYZImWt9oFFkpS20WdXSj0zR19Xk6TXXXnrAe2xvWA7Wt7vtp7OYWUgUrzrQWHYbQ3PeZoiMhziLtt+AdOQEFm2cboPpv0wV5AQVT21One+z6aTrRhbqzHsbLcBmM7uZ2noYznb/SqHNrhxuBAhN3a+B35ID677864iNzHEjnAd9R6IGfDOoNyg3W/MlC24t1QtHhCtcKFs9SiC8liQRQbwcshIZi2TPSj51YcslUPj/rm48PjIZMHaIXD6qcZEgpVbCh6KVYntUBwUcBNY7jmNK1guh3oUIYYrGuvFHm8jT/rlMXMjJUT8UBbt9min9k6ZquKf0KAAD//0oZDvg6AgAA
    objectset.rio.cattle.io/id: 4efe0994-2909-410a-81fa-7e11c55984b1
  creationTimestamp: "2023-08-16T14:05:12Z"
  generation: 1
  labels:
    app.kubernetes.io/component: mosip
  managedFields:
  - apiVersion: networking.istio.io/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:objectset.rio.cattle.io/applied: {}
          f:objectset.rio.cattle.io/id: {}
        f:labels:
          .: {}
          f:app.kubernetes.io/component: {}
      f:spec:
        .: {}
        f:gateways: {}
        f:hosts: {}
        f:http: {}
    manager: agent
    operation: Update
    time: "2023-08-16T14:05:12Z"
  name: superset
  namespace: superset
  resourceVersion: "9632622"
  uid: a8cfea5b-5fb6-4aa8-84d9-0db9f65d077b
spec:
  gateways:
  - superset
  hosts:
  - '*'
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: superset
        port:
          number: 8088 
```
