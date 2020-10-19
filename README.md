# thanos-prometheus-setup
This repository will describe how to set up Thanos across multiple Prometheus servers and k8s clusters, storing the metrics on S3/GCS and using grafana to display the collected data.

**⮕** The picture below show us a simplified version of what we intend to achieve.

![Image](images/thanos-prom-diagram.png?raw=true)

### What do you need to follow along with the instructions?

**⮕** One or more kubernetes clusters running (in this example only two clusters are going to be used)

**⮕** Object storage (s3/gcs will be used in this example)

### What is going to be used during this example?

**⮕** `kube-prometheus-stack` helm chart (https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

**⮕** `ingress-nginx-controller` helm chart  (https://github.com/kubernetes/ingress-nginx)

**⮕** A few manifest files from `kube-thanos` (https://github.com/thanos-io/kube-thanos)

***note: This is a manual process intended for learning purposes and can change. A terraform execution plan will come soon as well***

### Let's start (first kubernetes cluster)

`git clone git@github.com:danzarov/thanos-prometheus-setup.git`

`cd thanos-prometheus-setup`

**⮕** Adding the repositories we are going to need for prometheus, grafana, ingress controller, thanos, etc.

`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

`helm repo add ingress-repo https://kubernetes.github.io/ingress-nginx/`

**⮕** Install the ingress controller

`helm install ingress-nginx-controller ingress-repo/ingress-nginx`

**⮕** Creating a secret so the cluster can access the object storage (choose gcs or s3 depending on your setup)

### gcs example
`kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config-gcs.yaml --namespace default`

***note: if you are using the `thanos-storage-config-gcs.yaml` (google cloud storage) make sure to create a service account allowing access to storage***

### s3 example
`kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config-s3.yaml --namespace default`

**⮕** Install the kube-prometheus-stack

`helm install k1-prom prometheus-community/kube-prometheus-stack -f values/prometheus-thanos-values.yaml`

***note: feel free to modify the `prometheus-thanos-values.yaml` file. The file provided in this repository will make grafana public accessible with default password, so make sure to change it.***

**⮕** Make sure the thanos-sidecar can access the object storage by executing two commands below

`kubectl exec -it -n <prometheus with sidecar pod name> -c <sidecar container name> -- /bin/sh`

`thanos tools bucket ls --objstore.config="${OBJSTORE_CONFIG}"`

**⮕** Great! At this point we have `public access to grafana`, `prometheus collecting metrics` and `thanos-sidecar` sending data to **GCS/s3** 

### Create a new namespace to spin up thanos-store and thanos-querier

`kubectl create namespace thanos`

**⮕** Since the querier will be in the namespace `thanos` we need to create a secret in this namespace, so it can access the object storage

`kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config-s3.yaml --namespace thanos`

***note: make sure to choose the right file here, s3 or gcs depending on your setup***

`kubectl create -f manifests/`

**⮕** Now let's add thanos as data source for grafana

`kubectl describe pod <thanos query pod name> -n thanos` **⮕** `<thanos query pod name>`:9090 will be the pod that queries our object storage. You can use it as a data source for grafana.

### Let's do a similar approach on the second cluster (remember to switch contexts)

`kubectl config use-context <second-cluster-context>)`

**⮕** You can modify the `values/kube-thanos-values.yaml` file setting grafana enable false for example, since we can use the grafana from the first cluster to see the metrics from the object storage. Modify it the way it best fits your needs.

**⮕** Let's create a secret in the second cluster, so thanos can communicate to the object storage as well

`kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config-s3.yaml --namespace default`

**⮕** Install the kube-prometheus-stack

`helm install k2-prom prometheus-community/kube-prometheus-stack -f values/prometheus-thanos-values.yaml`

***note: I opted to not install ingress-nginx on the second cluster because I'm setting grafana enable to `false`*** 
***note: I opted to not create the manifest files on the second cluster because I'm only using thanos-querier in the first cluster to act as data source for grafana*** 

boom! you are done :) at this point you'll have two kubernetes clusters metrics being stored in the object storage and the grafana in the first cluster being able to analyze all the data.



