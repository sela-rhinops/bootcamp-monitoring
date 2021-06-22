# Lab 01: Deploy The Prometheus Operator

## Tasks

 - Configure your workspace
 - Deploy the Operator

## Configure your workspace

- The prometheus operator can be deployed by using the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart which deploys the prometheus operator as well as the whole prometheus stack already configured to monitor the Kubernetes cluster. However instead of using the helm chart we will be deploying each part separately and understand how each component works.

1. During the tutorial we will create several files that will be used to deploy each of the components of the Prometheus Stack. To make it easier let's create a new directory to be used as our workspace (we will use "~/monitoring-lab" but you can use a different one if you prefer)
  ```
  mkdir ~/monitoring-lab
  ```

## Deploy the Operator

- TODO: First download the bundle.yaml locally and sed namespace: default to namespace: monitoring

- To deploy the operator and CRDs we can use the official [bundle.yaml](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml) file.

1. With the default configuration it deploys the operator in the default namespace but we will change it to deploy it in the monitoring namespace that we will create. First download the file
```
sudo wget -P ~/monitoring-lab/operator https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
```

2. Replace "namespace: default" to "namespace: monitoring" in the ClusterRoleBinding, Deployment, ServiceAccount and Service of the operator in the bundle.yaml file we downloaded

3. Now we can use kubectl to create the monitoring namespace and apply the bundle.yaml file
  ```
  kubectl create ns monitoring
  kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml -n monitoring
  ```

4. After deploying the operator wait until the Operator's Pod is up and Running
  ```
  kubectl get po -w -n monitoring
  ```

5. To see the operator's logs use the following command
  ```
  kubectl logs -n monitoring prometheus-operator-<uuid>-<uuid>
  ```

6. We can see what CRDs are deployed in the cluster by using the following command:
  ```
  kubectl get crds
  ```