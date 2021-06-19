# Lab 04: Deploy Grafana and configure Dashboards

## Tasks

 - Configure your workspace
 - Create Grafana configuration files
 - Deploy Grafana
 - Configure Dashboards to visualize data from Prometheus

## Configure your workspace

1. Create a dedicated folder for your Grafana files

```
mkdir ~/monitoring-lab/grafana
```

## Create Grafana configuration files

1. Now let's create all the Kubernetes configuration files needed to deploy Grafana. The first file is Grafana itself (I will use vim, you can use a different editor)
```
vim ~/monitoring-lab/grafana/grafana.yaml
```

2. The content of the file should be the below:
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: grafana
    spec:
      serviceAccountName: grafana
      securityContext:
        fsGroup: 472
        runAsUser: 472
      initContainers:
      - name: init-chown-data
        image: "busybox:latest"
        imagePullPolicy: IfNotPresent
        securityContext:
          runAsUser: 0
        command: ["chown", "-R", "472:472", "/var/lib/grafana"]
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - name: storage
          mountPath: "/var/lib/grafana"
      containers:
      - name: grafana
        image: grafana/grafana:8.0.3
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config
          mountPath: "/etc/grafana/grafana.ini"
          subPath: grafana.ini
        - name: storage
          mountPath: "/var/lib/grafana"
        ports:
        - name: ui
          containerPort: 3000
          protocol: TCP
        env:
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              name: grafana
              key: admin-user
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana
              key: admin-password
        livenessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: ui
          initialDelaySeconds: 60
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /api/health
            port: ui
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: config
        configMap:
          name: grafana
      - name: storage
        persistentVolumeClaim:
          claimName: grafana
```

3. The above configuration expects a service account and a persistent volume claim that will be used by the grafana deployment for storage . Let's create the service account:
```
vim ~/monitoring-lab/grafana/service-account.yaml
```

4. The content of the file should be the below
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: monitoring
```
5. And the persistent volume claim:
```
vim ~/monitoring-lab/grafana/pvc.yaml
```

6. The content of the file should be the below
```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

7. Next we will be creating a configmap to store the grafana configuration file. This configmap will be mounted to the grafana pods
```
vim ~/monitoring-lab/grafana/configmap.yaml
```

8. The content of the file should be the below:
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana
  namespace: monitoring
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true
    
    [grafana_net]
    url = https://grafana.net
    
    [log]
    mode = console
    
    [paths]
    data = /var/lib/grafana/data
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
```

9. Now we are also going to create a secret that contains the username and password for the admin user.
```
vim ~/monitoring-lab/grafana/secret.yaml
```

10. The content of the file should be the below:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: grafana
  namespace: monitoring
type: Opaque
data:
  admin-user: YWRtaW4=
  admin-password: Y2hhbmdlbWU=
```
  - The username is "admin" and the password is "changeme". Feel free to change these values but remember to base64 encode them first.

11. We will also need a service to expose Grafana and access it's UI. For this tutorial we will be using a NodePort service
```
vim ~/monitoring-lab/grafana/service.yaml
```

12. The content of the file should be the below:
```
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
  - name: ui
    port: 3000
    protocol: TCP
  selector:
    app: grafana
```

13. Last let's create a ServiceMonitor to configure it as a target for prometheus.
```
vim ~/monitoring-lab/grafana/service-monitor.yaml
```

14. The content of the file should be the below:
```
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: grafana
  namespace: monitoring
  labels:
    team: devops-bootcamp
spec:
  endpoints:
  - honorLabels: true
    port: ui
  selector:
    matchLabels:
      app: grafana
```

## Deploy Grafana

1.  Use kubectl to deploy the Grafana with the configuration files created in the previous step.
```
kubectl apply -f ~/monitoring-lab/grafana/ -n monitoring
```

2. It may take up to a few minutes until all the resources are created and prometheus is ready for use. You can see the status of the pod with:
```
kubectl get po -n monitoring -w
```

3. You can also get an overview of all ServiceMonitors that were deployed using crds by using:
```
kubectl get servicemonitor --all-namespaces
```

4. Go to your browser and browse to:
```
http://<k8s-node-ip-or-dns>:<nodeport>
```

5. Log in with the admin user using the credentials that were stored as a secret. If these values were not changed you can login with:
```
username: admin
password: changeme
```

## Configure Dashboards to visualize data from Prometheus

1. Before we can go ahead and configure dashboards we need to define Prometheus as a Data Source. In the menu on the left hover the Settings icon and click on "Data Sources"
  ![grafana](/images/grafana.png)

2. Click on the "Add Data Source" button and select Prometheus
  ![grafana-2](/images/grafana-2.png)

3. The only thing we need to input here is the prometheus url that grafana will use to get data. Because we are running on top of kubernetes and we created a service for prometheus we can use the name of the service and it's port (prometheus-main:9090).
  ![grafana-3](/images/grafana-3.png)

4. Click on Save and Test to configure.

5. Now that we have Prometheus configured as a Data Source we are going to import our first dashboard. Hover the Dashboards icon and click on "Manage"
  ![grafana-4](/images/grafana-4.png)

6. Click on the "Import" Button then in the "Import via grafana.com" text box enter 1860 which is the ID of the [Node Exporter Full](https://grafana.com/grafana/dashboards/1860) dashboard from grafana.com and click on the "Load" Button
  ![grafana-5](/images/grafana-5.png)
  
7. Select Prometheus as your data source and click on the "Import" button
  ![grafana-6](/images/grafana-6.png)

8. This dashboard allows us to visualize the data that is being collected by node_exporter and stored in Prometheus. Take a few minutes to explore the different panels of the dashboard
  ![grafana-7](/images/grafana-7.png)

9. Repeat the steps 5 to 8 but use the following IDs:
```
Grafana Internals Dashboard: 3590
Prometheus Stats: 12134
```
- These dashboards allow us to visualize data coming from the Service Monitors that we configured to receive metrics from Prometheus and Grafana (Source: [Prometheus Stats](https://grafana.com/grafana/dashboards/12134), [Grafana Internals](https://grafana.com/grafana/dashboards/3590))
