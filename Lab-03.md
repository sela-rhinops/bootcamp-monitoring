# Lab 01: Deploy Node Exporter and configure Prometheus

## Tasks

 - Configure your workspace
 - Create Node Exporter configuration files
 - Deploy Node Exporter
 - View Node Exporter metrics in Prometheus

## Configure your workspace

1. Create a dedicated folder for your Node Exporter files

```
mkdir ~/monitoring-lab/node-exporter
```

## Create Node Exporter configuration files

- A Prometheus Exporter is a piece of software that. Can fetch statistics from another, non-Prometheus system. Can turn those statistics into Prometheus metrics, using a client library. Starts a web server that exposes a /metrics URL, and have that URL display the system metrics. In this lab we will be deploying the node_exporter which is just a binary that exposes all the server metrics through it's /metrics endpoint.

1. Now let's create all the Kubernetes configuration files needed to deploy Node Exporter. The first file is Node Exporter itself (I will use vim, you can use a different editor)

```
vim ~/monitoring-lab/node-exporter/node-exporter.yaml
```

2. The content of the file should be the below:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: node-exporter
      app.kubernetes.io/part-of: kube-prometheus
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: node-exporter
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 1.1.2
    spec:
      containers:
      - args:
        - --web.listen-address=127.0.0.1:9100
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --no-collector.wifi
        - --no-collector.hwmon
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)
        - --collector.netclass.ignored-devices=^(veth.*)$
        - --collector.netdev.device-exclude=^(veth.*)$
        image: quay.io/prometheus/node-exporter:v1.1.2
        name: node-exporter
        resources:
          limits:
            cpu: 250m
            memory: 180Mi
          requests:
            cpu: 102m
            memory: 180Mi
        volumeMounts:
        - mountPath: /host/sys
          # mountPropagation: HostToContainer
          name: sys
          readOnly: true
        - mountPath: /host/root
          # mountPropagation: HostToContainer
          name: root
          readOnly: true
      - args:
        - --logtostderr
        - --secure-listen-address=[$(IP)]:9100
        - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - --upstream=http://127.0.0.1:9100/
        env:
        - name: IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        image: quay.io/brancz/kube-rbac-proxy:v0.10.0
        name: kube-rbac-proxy
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: https
        resources:
          limits:
            cpu: 20m
            memory: 40Mi
          requests:
            cpu: 10m
            memory: 20Mi
        securityContext:
          runAsGroup: 65532
          runAsNonRoot: true
          runAsUser: 65532
      hostNetwork: true
      hostPID: true
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: node-exporter
      tolerations:
      - operator: Exists
      volumes:
      - hostPath:
          path: /sys
        name: sys
      - hostPath:
          path: /
        name: root
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 10%
    type: RollingUpdate
```
  -  By using the DaemonSet kubernetes object we are ensureing that there is one instance of node_exporter on all the nodes of the cluster.

3. The above configuration expects a service account with the necessary permissions required for monitoring in the kubernetes cluster. Let's create the service account:
```
vim ~/monitoring-lab/node-exporter/service-account.yaml
```

4. The content of the file should be the below:
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
  namespace: monitoring
```

5. Now let's create the cluster role:
```
vim ~/monitoring-lab/node-exporter/cluster-role.yaml
```
 
6. The content of the file should be the below:
```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
rules:
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
```

7. And the cluster role binding to associate the above role to the service account:
```
vim ~/monitoring-lab/node-exporter/cluster-role-binding.yaml
```

8. The content of the file should be the below:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-exporter
subjects:
- kind: ServiceAccount
  name: node-exporter
  namespace: monitoring
```

9. We will also need a ClusterIp service to internally expose node-exporter so that Prometheus can access it's /metrics endpoint.
```
vim ~/monitoring-lab/node-exporter/service.yaml
```

10. The content of the file should be the below
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: https
    port: 9100
    targetPort: https
  selector:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
```

11. Last we will need to create a ServiceMonitor in order to make the operator configure Node Exporter as a target in Prometheus.
```
vim ~/monitoring-lab/node-exporter/service-monitor.yaml
```

12. The content of the file should be the below
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    team: devops-bootcamp
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: node-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 1.1.2
  name: node-exporter
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 15s
    port: https
    relabelings:
    - action: replace
      regex: (.*)
      replacement: $1
      sourceLabels:
      - __meta_kubernetes_pod_node_name
      targetLabel: instance
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
  jobLabel: app.kubernetes.io/name
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: node-exporter
      app.kubernetes.io/part-of: kube-prometheus
```

## Deploy Node Exporter

1.  Use kubectl to deploy the Node Exporter with the configuration files created in the previous step.
```
kubectl apply -f ~/monitoring-lab/node-exporter/ -n monitoring
```

2. It may take up to a few minutes until all the resources are created and prometheus is ready for use. You can see the status of the pod with:
```
kubectl get po -n monitoring -w
```

3. You can also get an overview of all ServiceMonitors that were deployed using crds by using:
```
kubectl get servicemonitor --all-namespaces
```

## View Node Exporter metrics in Prometheus

1. Now that we deployed node-exporter and configured it as a target in Prometheus by using the ServiceMonitor CRD we should be able to see it configured as a target in Prometheus. In the browser go to http://<k8s-node-ip-or-dns>:<nodeport> then Status > Targets from the NavBar or you can simply browse to http://<k8s-node-ip-or-dns>:<nodeport>/targets

![node-exporter](/images/node-exporter.png)

2. Now we can go to Graph from the nav bar and execute the following PromQL query and click on the Execute button:
```
rate(node_network_receive_bytes_total[5m]) * 8
```

2. Click on the "Graph" tab to get a visualization of the data
![node-exporter-2](/images/node-exporter-2.png)
  - This graph will show us the number of bits received over the various network interfaces per second
  - For more info on PromQL check the [PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/) or this [Post](https://valyala.medium.com/promql-tutorial-for-beginners-9ab455142085) 