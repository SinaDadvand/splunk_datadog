# Datadog Local Lab (Agent + Docker + Kind + ArgoCD)

Hands-on guide to run the Datadog Agent locally in Docker, collect logs/metrics from containers and Kind, and view them in Datadog. You need a Datadog account and API key.

Commands target Windows PowerShell (v5.1+) with Docker Desktop running.

## What you’ll learn
- Run the Datadog Agent locally and understand core features (metrics, logs, APM)
- Collect Docker container telemetry via the Agent
- Deploy the Agent to a local Kind cluster with Helm
- Explore logs, metrics, and (optional) traces in the Datadog UI

## Prerequisites
- Datadog account + API key
- Windows 10/11, Docker Desktop, PowerShell 5.1+
- Kind, kubectl, Helm (see `kind-argocd-setup.md`)

Notes for Windows/Docker Desktop:
- The Agent needs access to the Docker socket for container metadata.
- Ensure outbound network access to your Datadog site domain (e.g., us5.datadoghq.com).

---

## 1) Environment

What/Why:
- Provide your API key and site so the Agent can send data to your account.

Set environment variables (replace with your values):
```
$env:DD_API_KEY = "<YOUR_DD_API_KEY>"
$env:DD_SITE    = "us5.datadoghq.com"  # or us3.datadoghq.com, datadoghq.eu, etc.
```

How to verify:
- Echo the variables: `$env:DD_API_KEY.Substring(0,6) + '...'`, `$env:DD_SITE`

Expected result:
- Variables are set for your current PowerShell session.

---

## 2) Run Datadog Agent with Docker

What/Why:
- Start a single Datadog Agent container that will auto-discover other containers and collect metrics/logs. APM is enabled on port 8126.

Run the agent:
```
docker run -d --name datadog-agent `
  -e DD_API_KEY=$env:DD_API_KEY `
  -e DD_SITE=$env:DD_SITE `
  -e DD_APM_ENABLED=true `
  -e DD_LOGS_ENABLED=true `
  -e DD_CONTAINER_EXCLUDE="name:datadog-agent" `
  -v /var/run/docker.sock:/var/run/docker.sock:ro `
  -v /proc/:/host/proc/:ro `
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro `
  -p 8126:8126 `
  gcr.io/datadoghq/agent:7
```

How to verify:
- Check logs: `docker logs -f datadog-agent` (look for “initialized” and no auth errors).
- Start a sample container to generate signals:
```
docker run -d --name web -p 8082:80 nginx
for ($i=0; $i -lt 20; $i++) { Invoke-WebRequest -UseBasicParsing http://localhost:8082 | Out-Null }
```

What to see in Datadog:
- Infrastructure → Containers: a list including `web` and `datadog-agent`.
- Logs → Live Tail/Explorer: container logs for `web` over time.

Expected result:
- Within a few minutes, the container appears with CPU/memory metrics. Logs start flowing when `DD_LOGS_ENABLED=true` and the integration detects stdout/stderr.

---

## 3) Kind: deploy Agent as DaemonSet

What/Why:
- Run the Agent on each node via DaemonSet to collect Kubernetes logs/metrics; enable APM and process collection for richer telemetry.

If you haven’t created Kind yet, follow `kind-argocd-setup.md` first.

Install with Helm:
```
kubectl create namespace datadog
helm repo add datadog https://helm.datadoghq.com
helm repo update

helm upgrade --install datadog datadog/datadog `
  -n datadog `
  --set datadog.apiKey=$env:DD_API_KEY `
  --set datadog.site=$env:DD_SITE `
  --set datadog.logs.enabled=true `
  --set datadog.apm.enabled=true `
  --set datadog.processAgent.enabled=true `
  --set targetSystem=linux
```

How to verify:
- `kubectl -n datadog get pods -o wide` (all pods Running)
- `kubectl -n datadog logs deploy/datadog` to check for errors

Generate workload in Kind:
```
kubectl create namespace demo
kubectl -n demo create deploy web --image=nginx --replicas=2
kubectl -n demo expose deploy/web --port 80 --type ClusterIP
kubectl -n demo get pods -o wide
```

What to see in Datadog:
- Infrastructure → Kubernetes: cluster “kind-dev” (or your cluster name), nodes, pods
- Logs → search `kube_namespace:demo`
- Metrics → filter by `kube_cluster_name` label

Expected result:
- Kubernetes entities appear with node/pod health and logs. If not, ensure network egress and that your site/API key are correct.

---

## 4) Sample app with logs and traces (optional APM)

What/Why:
- NGINX provides logs/metrics. For traces, you need an instrumented app that sends data to the Agent at port 8126.

If you deploy an instrumented app (e.g., Node, Python, .NET), set:
- DD_AGENT_HOST to the Agent’s hostname or service (K8s) or 127.0.0.1 (sidecar pattern)
- DD_TRACE_AGENT_PORT=8126

Quick traffic to NGINX (logs/metrics only):
```
for ($i=0; $i -lt 50; $i++) { Invoke-WebRequest -UseBasicParsing http://web.demo.svc.cluster.local -TimeoutSec 2 | Out-Null }
```

What to see in Datadog:
- APM → Services: only if you sent traces from an instrumented app
- Logs: `kube_namespace:demo`
- Dashboards: Container and Kubernetes overview dashboards

Expected result:
- Logs/metrics visible. Traces appear only when your app is instrumented.

---

## 5) ArgoCD GitOps option

What/Why:
- Manage the Datadog Helm release declaratively via ArgoCD so changes flow from Git.

Steps:
- Put a `values.yaml` in a Git repo with your overrides (apiKey, site, logs/apm/process flags).
- In ArgoCD, create an Application referencing chart `datadog/datadog` and your `values.yaml`.

Verify/Expected:
- ArgoCD shows the `datadog` app syncing and Health=Healthy. `kubectl -n datadog get pods` shows the Agent running.

---

## 6) Clean up
```
docker rm -f datadog-agent
helm uninstall datadog -n datadog
kubectl delete ns demo
kubectl delete ns datadog
```

Expected result:
- Local Agent stopped, Kubernetes DaemonSet removed, demo resources deleted.

---

## Troubleshooting and tips
- Auth errors: Check API key and site; `docker logs -f datadog-agent` or `kubectl -n datadog logs`.
- No containers in Infra view: Ensure Docker socket is mounted and Agent is running.
- No K8s data: Confirm cluster connectivity and that Helm release is in the right namespace; check network egress to Datadog.
- Logs missing: Confirm `datadog.logs.enabled=true` and look for exclusions via `DD_CONTAINER_EXCLUDE`.
