# Lab 05: Deploy Alertmanager and configure Alerts

## Tasks

 - Configure your workspace
 - Create Alertmanager configuration files
 - Deploy Alertmanager using CRDs
 - Create Rules
 - View Alerts

## Configure your workspace

1. Create a dedicated folder for your Alertmanager files

```
mkdir ~/monitoring-lab/alertmanager
```

## Create Alertmanager configuration files

1. Now let's create all the Kubernetes configuration files needed to deploy Alertmanager. The first file is Alertmanager itself (I will use vim, you can use a different editor)
```
vim ~/monitoring-lab/alertmanager/alertmanager.yaml
```

2. The content of the file should be the below:
```
---
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: demo
  namespace: monitoring
spec:
  version: v0.22.2
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 300m
      memory: 512Mi

```
  - We are using the "kind" Alertmanager which is also a CRD that is managed by the operator.

3. Now let's create a secret for the alertmanager configuration. This secret will be captured by the operator and it will then configure Alertmanager. With this configuration we will not actually receive alerts but we will be able to see the flow.

```
vim ~/monitoring-lab/alertmanager/config.yaml
```

4. The content of the file should be the below:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-demo
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |-
    global:
      resolve_timeout: 5m
      smtp_smarthost: smtp.gmail.com:587
      smtp_from: email@mail.com
      smtp_auth_username: email@mail.com
      smtp_auth_identity: email@mail.com
      smtp_auth_password: password
      smtp_require_tls: true
    route:
      group_by:
      - cluster
      - alertname
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: email
      routes:
      - receiver: email
        match:
          severity: warning
    receivers:
      - name: email
        email_configs:
        - to: email@mail.com
```


5. Another way of configuring Alertmanager is by using the "AlertmanagerConfig" CRD. This CRD will be captured by the operator and configure Alertmanger. It is possible to use both methods together. Let's create the AlertmanagerConfig yaml:
```
vim  ~/monitoring-lab/alertmanager/alertmanagerconfig.yaml
```

6. The content of the file should be the below:
```
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config-example
  labels:
    alertmanagerConfig: example
spec:
  route:
    groupBy: ['job']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: 'wechat-example'
  receivers:
  - name: 'wechat-example'
    wechatConfigs:
    - apiURL: 'http://wechatserver:8080/'
      corpID: 'wechat-corpid'
      apiSecret:
        name: 'wechat-config'
        key: 'apiSecret'
---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: wechat-config
data:
  apiSecret: d2VjaGF0LXNlY3JldAo=
```

7. We will also need a Service to be able to access the alertmanager UI, for this workshop we will be using a NodePort servive
```
vim ~/monitoring-lab/alertmanager/service.yaml
```

8. The content of the file should be the below
```
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-main
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9093
    protocol: TCP
    targetPort: web
  selector:
    alertmanager: demo
```

9. And we will also configure alertmanager as a target for prometheus by using the ServiceMonitor CRD
```
vim ~/monitoring-lab/alertmanager/service-monitor.yaml
```

10. The content of the file should be the below
```
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: alertmanager
  namespace: monitoring
  labels:
    team: devops-bootcamp
spec:
  endpoints:
  - interval: 15s
    port: web
    scrapeTimeout: 15s
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      operated-alertmanager: "true"
```

## Deploy Alertmanager using CRDs

1. Use kubectl to deploy Alertmanager with the configuration files created in the previous step.
```
kubectl apply -f ~/monitoring-lab/alertmanager/ -n monitoring
```

2. It may take up to a few minutes until all the resources are created and alertmanager is ready for use. You can see the status of the pod with:
```
kubectl get po -n monitoring -w
```

3. You can also get an overview of all alertmanager instances that were deployed using crds by using:
```
kubectl get alertmanager --all-namespaces
```

4. Access the logs of the Prometheus pod:
```
kubectl logs -n monitoring alertmanager-demo-0 alertmanager
```

## Create Rules

1. Before we take a look at the UI of alertmanager let's use another CRD called "PrometheusRule" that will allow us to configure alerts and maintain them by using yaml files and kubernetes. Let's create the yaml file:
```
vim  ~/monitoring-lab/alertmanager/demo-rules.yaml
```

2. The content of the file should be the below:
```
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: general
  namespace: monitoring
  labels:
    team: devops-bootcamp
spec:
  groups:
  - name: general
    rules:
    - alert: TargetDown
      annotations:
        message: '{{ $value }}% of the {{ $labels.job }} targets are down.'
      expr: 100 * (count(up == 0) BY (job) / count(up) BY (job)) > 10
      for: 1m
      labels:
        severity: warning
    - alert: DeadMansSwitch-serviceprom
      annotations:
        description: This is a DeadMansSwitch meant to ensure that the entire Alerting
          pipeline is functional.
        summary: Alerting DeadMansSwitch
      expr: vector(1)
      labels:
        severity: none
```
- The first rule will let us know if there a target of prometheus is down and the second one is an alert that will always fire so that we can see if the alerting flow is working correctly. For more info on alerts see the [Official documentation](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) and for some well known examples you can use [THIS](https://awesome-prometheus-alerts.grep.to/rules.html) site.

3. Let's use kubectl and apply the above configuration
```
kubectl apply -f ~/monitoring-lab/alertmanager/demo-rules.yaml
```
4. To see the alerts that were deployed using CRDs we can use:
```
kubectl get prometheusrules -n  --all-namespaces
```

## View Alerts

1. Now that we configured some rules, the operator will get the CRDs and configure Prometheus. We can see what rules are configured in the Prometheus UI. Browse to:
```
http://<k8s-node-ip-or-dns>:<nodeport-of-prometheus>/rules
```
![alertmanager](/images/alertmanager-1.png)
  - Here we can see the state of the alerts, if there is an error, when was the last time that prometheus evalueted the rules and how much time it took.

2. Prometheus will evaluate the expresion in the Alert aganist the data that it has stored. Wenever it returns a value for more than the time specified in the alert it will then fire and Prometheus will notify Alertmanager that it has to send an alert. Let's browse to Alerts and you will see that there is an alert in Firing state, this alert comes from the rule that we configured in previous steps.
![alertmanager](/images/alertmanager-2.png)

3. If we browse to the AlertManager UI you will see that it has received the alert
![alertmanager](/images/alertmanager-3.png)