# Lab 05: Deploy Alertmanager and configure Alerts

## Tasks

 - Configure your workspace
 - Create Alertmanager configuration files
 - Deploy Alertmanager
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

3. Now let's create a secret for the alertmanager configuration. Tis secret will be captured by the operator and it will then configure Alertmanager.

```
vim ~/monitoring-lab/alertmanager/config.yaml
```

