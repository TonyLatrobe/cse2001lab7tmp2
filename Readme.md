# cse2001lab7: Observability Lab (Jenkins, Prometheus, Grafana, Loki, Alloy)

This lab runs Jenkins, a demo app and a full observability stack in one Kubernetes (MicroK8s) namespace, `observability`. A failing Jenkins pipeline sends its error to Loki, and Grafana shows pipeline, app metrics and logs on one dashboard.

## Architecture

```
Git repo (Jenkinsfile)
       │
       ▼
Jenkins pipeline -----> kubectl apply demo-app ---> demo-app pod
       │                                    \
       │                                     \
Prometheus <----- metrics         Alloy <---- logs
       │                                     \
       │                                      \
       │                                     Loki
       │                                       │
       ▼                                       ▼
   Grafana dashboard (pipeline + app metrics + logs)
```

## Project structure

```
.
├── Jenkinsfile
├── Readme.md
├── k8s/          namespace, Jenkins (sa, rbac, deployment, service, pvc, ca configmap)
├── demo-app/     nginx demo app (configmap, deployment, service)
├── prometheus/   config, alert rules, deployment, service
├── grafana/      deployment, service, datasources configmap
├── logging/      Loki (config, deployment, service) and Alloy (config, service, sa, rbac, daemonset)
├── chaos/        chaos-pods.yaml (intentionally triggers alerts)
└── dashboards/   observability-dashboard.json (import into Grafana)
```

## 1. Apply the Kubernetes manifests

```bash
# 1. Namespace first
sudo microk8s kubectl apply -f k8s/namespace.yaml

# 2. Storage + RBAC
sudo microk8s kubectl apply -f k8s/jenkins-sa.yaml
sudo microk8s kubectl apply -f k8s/jenkins-rbac.yaml
sudo microk8s kubectl apply -f k8s/jenkins-pvc.yaml
sudo microk8s kubectl apply -f k8s/jenkins-ca-configmap.yaml

# 3. Logging stack (Loki before Alloy: Alloy needs somewhere to push)
sudo microk8s kubectl apply -f logging/loki-config.yaml
sudo microk8s kubectl apply -f logging/loki-deployment.yaml
sudo microk8s kubectl apply -f logging/loki-service.yaml
sudo microk8s kubectl apply -f logging/alloy-sa.yaml
sudo microk8s kubectl apply -f logging/alloy-rbac.yaml
sudo microk8s kubectl apply -f logging/alloy-service.yaml
sudo microk8s kubectl apply -f logging/alloy-config.yaml
sudo microk8s kubectl apply -f logging/alloy-daemonset.yaml

# 4. Observability
sudo microk8s kubectl apply -f prometheus/
sudo microk8s kubectl apply -f grafana/

# 5. App + CI
sudo microk8s kubectl apply -f demo-app/
sudo microk8s kubectl apply -f k8s/jenkins-deployment.yaml
sudo microk8s kubectl apply -f k8s/jenkins-service.yaml
```

## 2. Verify

```bash
# All pods running
sudo microk8s kubectl get pods -n observability

# Loki healthy
sudo microk8s kubectl logs deploy/loki -n observability | grep "started"

# Alloy shipping logs
sudo microk8s kubectl logs daemonset/alloy -n observability | grep "loki"

# Demo app calling itself (watch nginx access logs)
sudo microk8s kubectl logs deploy/demo-app -n observability -f
```

| Service | URL |
|---|---|
| Demo app | http://127.0.0.1:30080/ |
| Prometheus (all targets should be UP) | http://127.0.0.1:30334 |
| Grafana (admin/admin) | http://127.0.0.1:30300/ |
| Jenkins | http://127.0.0.1:32043 |

In Grafana, go to Explore → Loki and add these queries (add one for each pod):

```
count_over_time({namespace="observability", pod=~"demo-app-.*"} |= " 200 "[1m])

count_over_time({namespace="observability", pod=~"demo-app-.*"} |= " 404 "[1m])
```

## 3. Grafana dashboard

Import `dashboards/observability-dashboard.json` via Grafana → Dashboards → Import → Upload JSON.

## 4. Jenkins

Get the initial admin password:

```bash
sudo microk8s kubectl exec -it deploy/jenkins -n observability -- cat /var/jenkins_home/secrets/initialAdminPassword
```

Log in as `admin` at http://127.0.0.1:32043 (bridged network: `http://<VM_IP>:32043`).

- Jenkins URL: `http://jenkins.observability.svc.cluster.local:8080/`
- Kubernetes cloud: name `KubernetesCloud`, Kubernetes URL `https://kubernetes.default.svc`, Jenkins URL `http://jenkins.observability.svc.cluster.local:8080`, tick "Disable https certificate check"
- Pod template: name `pipelinepod`, container `jnlp`, image `localhost:32000/jenkins-inbound-agent:lab`, working directory `/home/jenkins/agent`, pod service account `jenkins`
- Pipeline: New Item → Pipeline → "Pipeline script from SCM", Git, repository `https://github.com/<username>/cse2001lab7`, branch `*/main`, script path `Jenkinsfile`, tick Lightweight checkout. Save, then Build Now.

The Jenkinsfile runs `make build`, which always fails because there is no makefile. On failure it pushes the error to Loki and fails the build, which shows up in the Grafana "Jenkins Error Logs" panel.

## 5. Chaos (optional)

Trigger alerts on purpose, then turn the chaos pods off:

```bash
sudo microk8s kubectl apply -f chaos/chaos-pods.yaml
sudo microk8s kubectl delete -f chaos/chaos-pods.yaml
```

## Clean up

```bash
sudo microk8s kubectl delete -f chaos/chaos-pods.yaml
sudo microk8s kubectl delete namespace observability
```

This removes the observability namespace and frees up memory for the next lab.
