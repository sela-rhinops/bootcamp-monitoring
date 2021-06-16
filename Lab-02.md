# Lab 01: Deploy Prometheus using the Operator and CRDs

## Tasks

 - Configure your workspace
 - Create Prometheus configuration files
 - Deploy Prometheus using CRDs
 - Explore Prometheus

## Configure your workspace

1. Create a dedicated folder for your prometheus files

```
mkdir ~/monitoring-lab/prometheus
```

## Create Prometheus configuration files

1. Now let's create all the Kubernetes configuration files needed to deploy Prometheus. The first file is Prometheus itself (I will use vim, you can use a different editor)

```
vim ~/monitoring-lab/prometheus/prometheus.yaml
```

2. The content of the file should be the below:

```
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: demo
  namespace: monitoring
spec:
  version: v2.25.0
  alerting:
    alertmanagers:
    - namespace: monitoring
      name: alertmanager-operated
      port: web
  serviceMonitorSelector:
    matchLabels:
      team: devops-bootcamp
  ruleSelector:
    matchLabels:
      team: devops-bootcamp
  serviceAccountName: prometheus
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  enableAdminAPI: false
  storage:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 20Gi
  securityContext:
    fsGroup: 0
    runAsNonRoot: false
    runAsUser: 0
  replicas: 1
  retention: 7d
  scrapeInterval: 30s
```
As you may have noticed the "kind" of this object is Prometheus. This is one of the Custom Resource Definitions (CRD) that were installed as part of the operator's installation

3. The above configuration expects a service account with the necessary permissions required for monitoring in the kubernetes cluster. Let's create the service account:
```
vim ~/monitoring-lab/prometheus/service-account.yaml
```

4. The content of the file should be the below:
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
```

5. Now let's create the cluster role:
```
vim ~/monitoring-lab/prometheus/cluster-role.yaml
```
 
6. The content of the file should be the below:
```
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

7. And the cluster role binding to associate the above role to the service account:
```
vim ~/monitoring-lab/prometheus/cluster-role-binding.yaml
```

8. The content of the file should be the below:
```
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

9. Last we need a service to expose Prometheus. In this tutorial we will be using NodePort
```
vim ~/monitoring-lab/prometheus/service.yaml
```

10. The content of the file should be the below
```
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-main
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: demo
```

## Deploy Prometheus using CRDs

1. Use kubectl to deploy the Prometheus with the configuration files created in the previous step.
```
kubectl apply -f ~/monitoring-lab/prometheus/ -n monitoring
```

2. It may take up to a few minutes until all the resources are created and prometheus is ready for use. You can see the status of the pod with:
```
kubectl get po -n monitoring -w
```

3. You can also get an overview of all prometheus instances that were deployed using crds by using:
```
kubectl get prometheus --all-namespaces
```

4. Access the logs of the Prometheus pod:
```
kubectl logs -n monitoring prometheus-demo-0 prometheus
```

## Explore Prometheus

1. Browse to your Prometheus instance from any browser
```
http://<k8s-node-ip-or-dns>:<nodeport>
```
![Prometheus-ui](/images/prometheus-ui.png)

2. You can see the current prometheus configuration under "Status/Configuration" and the prometheus information in "Status/Runtime & Build Information"

![Prometheus-ui-2](/images/prometheus-ui-2.png)

3. Right now prometheus has no targets to gather metrics from. Instead of configuring targets the tradional way (by creating configmaps manualy) we are going to use "ServiceMonitor" which is another CRD that was installed as along the operator. The operator will receive the ServiceMonitor and configure the targets for prometheus. Let's create a ServiceMonitor for prometheus itself:
```
vim ~/monitoring-lab/prometheus/service-monitor.yaml
```

4. The content of the file should be the below:
```
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    team: devops-bootcamp
spec:
  endpoints:
  - honorLabels: true
    port: web
  selector:
    matchLabels:
      operated-prometheus: "true"
```

5. Apply the ServiceMonitor:
```
kubectl apply -f ~/monitoring-lab/prometheus/service-monitor.yaml
```

6. To see what ServiceMonitors are deployed in the cluster you can use:
```
kubectl get servicemonitor --all-namespaces
```

7. In your browser head over to Status > Targets and you will see that a new target was added

![Prometheus-ui-3](/images/prometheus-ui-3.png)

8. Now that we have some data let's run a query. Go to "Graph" and write the below in the "Expression" field and click "Execute"
```
scrape_duration_seconds
```
![Prometheus-ui-4](/images/prometheus-ui-4.png)

9. Right now you are seeing the time it took to Prometheus to scrape (collect) it metrics the last time, but let's click on "Graph" to see the historical data

![Prometheus-ui-5](/images/prometheus-ui-5.png)