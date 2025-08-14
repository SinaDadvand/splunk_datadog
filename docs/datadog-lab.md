# Datadog Local Lab (Agent + Docker + Kind + ArgoCD)

Hands-on guide to run the Datadog Agent locally in Docker, collect logs/metrics from containers
**How to verify the agent is working:**
```powershell
# Check agent **How to verify the agent is working:**
```powershell
# Step 1: Check that the agent container is running
docker ps | Select-String "datadog-agent"

# Step 2: Check agent logs for successful startup (look for success messages)
docker logs datadog-agent | Select-String "Datadog Agent is now running"

# Step 3: Look for any authentication errors
docker logs datadog-agent | Select-String -Pattern "ERROR|authentication"

# Step 4: Check detailed agent status
docker exec datadog-agent agent status
```

**Create a sample container to generate data:**
```powershell
# Start an NGINX web server
docker run -d --name web -p 8082:80 nginx

# Verify the web container is running and accessible
docker ps | Select-String "web"
Invoke-WebRequest -UseBasicParsing http://localhost:8082

# Generate some HTTP traffic to create metrics and logs
for ($i=0; $i -lt 20; $i++) { 
    try {
        Invoke-WebRequest -UseBasicParsing http://localhost:8082 | Out-Null
        Write-Host "Request $($i+1) completed"
    } catch {
        Write-Host "Request $($i+1) failed: $($_.Exception.Message)"
    }
    Start-Sleep 1
}
```

**Step-by-Step: How to View Data in Datadog UI**

**Step 1: Access Datadog**
1. Open your web browser
2. Navigate to your Datadog site (e.g., `https://us5.datadoghq.com`)
3. Log in with your credentials

**Step 2: View Container Infrastructure**
1. In the left navigation menu, click **Infrastructure**
2. Select **Containers** from the submenu
3. **What you should see:**
   - A list of containers including `web` and `datadog-agent`
   - Each container shows real-time CPU, memory, and network metrics
   - Status indicators showing if containers are running
4. **To explore further:**
   - Click on the `web` container name to see detailed metrics
   - Look for graphs showing CPU usage, memory consumption, and network activity
   - Notice the increase in metrics after generating HTTP traffic

**Step 3: View Container Logs**
1. In the left navigation menu, click **Logs**
2. Select **Live Tail** for real-time logs, or **Explorer** for historical logs
3. **What you should see:**
   - Real-time logs from your containers as they're generated
   - NGINX access logs from the `web` container showing HTTP requests
   - Timestamps, log levels, and container metadata
4. **To filter logs:**
   - In the search bar, type: `container_name:web` to see only NGINX logs
   - Try: `container_name:datadog-agent` to see agent logs
   - Use: `status:error` to filter for error logs only

**Step 4: View Host Map**
1. In the left navigation menu, click **Infrastructure**
2. Select **Host Map** from the submenu
3. **What you should see:**
   - A visual representation of your Docker host
   - Containers represented as boxes on the host
   - Color coding based on resource utilization (green = healthy, red = high usage)
4. **To interact:**
   - Hover over container boxes to see quick metrics
   - Click on a container to see detailed information

**Step 5: Explore Dashboards**
1. In the left navigation menu, click **Dashboards**
2. Look for pre-built dashboards:
   - Search for "Docker" to find Docker-related dashboards
   - Try "Container Overview" dashboard if available
3. **What you should see:**
   - Graphs showing container metrics over time
   - System resource utilization
   - Container counts and status information

**Step 6: Set Up Alerts (Optional)**
1. In the left navigation menu, click **Monitors**
2. Click **New Monitor** to create alerts
3. Example: Create an alert for high container CPU usage
4. **What you can monitor:**
   - Container CPU usage > 80%
   - Container memory usage > 90%
   - Container restart events

**Expected timeline for data appearance:**
- **Container discovery:** 30-60 seconds after agent starts
- **Basic metrics:** 2-3 minutes for first data points
- **Logs:** 1-2 minutes after containers start producing logs
- **Full dashboard data:** 5-10 minutes for complete visibility

**What each view tells you:**
- **Infrastructure → Containers:** Real-time resource usage and container health
- **Logs:** What your applications are doing and any errors
- **Host Map:** Overall system health and resource distribution
- **Dashboards:** Historical trends and patterns in your infrastructurehy:
- Run the Agent on each node via DaemonSet to collect Kubernetes logs/metrics; enable APM and process collection for richer telemetry.

**What we're learning:** How to deploy the Datadog Agent to a Kubernetes cluster using Helm to monitor Kubernetes workloads, pods, and cluster metrics.

**Why this matters:** In production environments, applications run in Kubernetes clusters. The Datadog Agent deployed as a DaemonSet ensures that every node in the cluster has monitoring capabilities, providing comprehensive visibility into Kubernetes-specific metrics, logs, and application performance.

**Key concepts:**
- **DaemonSet:** Ensures one agent pod runs on every Kubernetes node
- **Helm chart:** Packaged deployment configuration for complex applications
- **Kubernetes monitoring:** Cluster metrics, pod lifecycle, resource utilization
- **Process monitoring:** Visibility into processes running inside containers

**Prerequisites:** If you haven't created a Kind cluster yet, follow the instructions in `kind-argocd-setup.md` first.

**Where to run:** In your PowerShell terminal, with kubectl configured for your Kind cluster.successful startup
docker logs datadog-agent | Select-String "Datadog Agent is now running"

# Look for any authentication errors
docker logs datadog-agent | Select-String -Pattern "ERROR|authentication"

# Check agent status (should show "Agent status: Running")
docker exec datadog-agent agent status
```

**Create a sample container to generate data:**
```powershell
# Start an NGINX web server
docker run -d --name web -p 8082:80 nginx

# Generate some HTTP traffic to create metrics and logs
for ($i=0; $i -lt 20; $i++) { 
    try {
        Invoke-WebRequest -UseBasicParsing http://localhost:8082 | Out-Null
        Write-Host "Request $($i+1) completed"
    } catch {
        Write-Host "Request $($i+1) failed: $($_.Exception.Message)"
    }
    Start-Sleep 1
}
```

**What to expect in the Datadog UI (wait 2-3 minutes for data to appear):**

1. **Navigate to Infrastructure → Containers:**
   - You should see containers listed including `web` and `datadog-agent`
   - Each container shows CPU, memory, and network metrics
   - Click on a container to see detailed metrics and metadata

2. **Navigate to Logs → Live Tail:**
   - You should see real-time logs from your containers
   - NGINX access logs from the web container
   - Filter by container name to focus on specific logs

3. **Navigate to Infrastructure → Host Map:**
   - Shows your Docker host with running containers
   - Color-coded by resource utilization

**Expected results:**
- ✅ Agent container running: `docker ps` shows `datadog-agent` container
- ✅ No authentication errors in agent logs
- ✅ Web container accessible at http://localhost:8082
- ✅ Container metrics visible in Datadog UI within 2-3 minutes
- ✅ Container logs flowing to Datadog Logs section

**Troubleshooting:**
- **Agent won't start:** Check API key and site configuration
- **No containers visible:** Ensure Docker socket is properly mounted
- **No logs appearing:** Verify `DD_LOGS_ENABLED=true` and check container exclusions
- **Network issues:** Confirm outbound access to your Datadog site view them in Datadog. This lab is designed for first-time Datadog users who want to understand the platform's core capabilities through practical examples.

You need a Datadog account and API key (instructions provided below).

Commands target Windows PowerShell (v5.1+) with Docker Desktop running.

## What you'll learn
- **Run the Datadog Agent locally** and understand core features (metrics, logs, APM)
- **Collect Docker container telemetry** via the Agent's auto-discovery capabilities
- **Deploy the Agent to a local Kind cluster** with Helm for Kubernetes monitoring
- **Explore logs, metrics, and traces** in the Datadog UI with real-world examples
- **Understand Datadog's architecture** and how it differs from traditional logging platforms

## Datadog vs Splunk: Key Differences

Before diving into the lab, it's helpful to understand how Datadog differs from Splunk, especially if you're coming from a Splunk background:

| Aspect | Datadog | Splunk |
|--------|---------|--------|
| **Architecture** | Agent-based SaaS platform with centralized cloud processing | On-premises or cloud with indexers, forwarders, and search heads |
| **Data Collection** | Lightweight agents auto-discover and push metrics/logs/traces | Heavy forwarders or HEC endpoints pull/receive data |
| **Primary Focus** | Infrastructure monitoring, APM, and observability | Log analysis, SIEM, and search-driven analytics |
| **Query Language** | Custom query language with visual dashboards | SPL (Search Processing Language) |
| **Deployment** | Minimal agent deployment, SaaS backend | Full platform deployment and maintenance |
| **Data Storage** | Managed cloud storage with retention policies | Self-managed indexes with configurable retention |
| **Real-time Monitoring** | Built-in alerting and real-time dashboards | Custom searches and scheduled alerts |
| **APM Integration** | Native APM with distributed tracing | Requires additional configuration or third-party tools |

**Key Takeaway**: Datadog is designed for modern, cloud-native observability with minimal setup, while Splunk excels at deep log analysis and custom data investigations. Datadog agents automatically collect and correlate metrics, logs, and traces, whereas Splunk requires more manual configuration for similar functionality.

## Getting Started: Datadog Account Setup

**Step 1: Create a Datadog Account**
1. Go to [datadoghq.com](https://datadoghq.com) and sign up for a free 14-day trial
2. Choose your region (this determines your Datadog site URL)
3. Complete the account setup process

**Step 2: Get Your API Key**
1. After logging in, navigate to **Organization Settings** → **API Keys**
2. Click **New Key** to create a new API key, or copy an existing one
3. Give it a descriptive name like "Local Lab"
4. Copy the API key value - you'll need this for the lab

**Step 3: Note Your Datadog Site**
Your Datadog site URL depends on your region:
- **US1**: `datadoghq.com`
- **US3**: `us3.datadoghq.com` 
- **US5**: `us5.datadoghq.com`
- **EU**: `datadoghq.eu`
- **AP1**: `ap1.datadoghq.com`

You can find your site in the URL when logged into Datadog (e.g., `https://us5.datadoghq.com/`).ocal Lab (Agent + Docker + Kind + ArgoCD)

Hands-on guide to run the Datadog Agent locally in Docker, collect logs/metrics from containers and Kind, and view them in Datadog. You need a Datadog account and API key.

Commands target Windows PowerShell (v5.1+) with Docker Desktop running.

## What you’ll learn
- Run the Datadog Agent locally and understand core features (metrics, logs, APM)
- Collect Docker container telemetry via the Agent
- Deploy the Agent to a local Kind cluster with Helm
- Explore logs, metrics, and (optional) traces in the Datadog UI

## Prerequisites
- **Datadog account + API key** (instructions provided above)
- **Windows 10/11** with Docker Desktop installed and running
- **PowerShell 5.1+** (comes with Windows)
- **Kind, kubectl, Helm** for Kubernetes examples (see `kind-argocd-setup.md`)

**Important Notes for Windows/Docker Desktop:**
- The Agent needs access to the Docker socket (`/var/run/docker.sock`) for container metadata
- Ensure outbound network access to your Datadog site domain (e.g., `us5.datadoghq.com`)
- Docker Desktop must be running in Linux containers mode (not Windows containers)

---

## 1) Environment Setup

**What we're learning:** How to configure authentication for the Datadog Agent to communicate with your Datadog account.

**Why this matters:** The Datadog Agent requires an API key and site configuration to send telemetry data to the correct Datadog instance. Without proper authentication, the agent will fail to start or send data.

**Where to run these commands:** In your PowerShell terminal, from any directory.

Set environment variables (replace with your actual values from the setup above):
```powershell
$env:DD_API_KEY = "<YOUR_DD_API_KEY>"     # Replace with your actual API key
$env:DD_SITE    = "us5.datadoghq.com"    # Replace with your Datadog site
```

**How to verify the setup:**
```powershell
# Check that variables are set (shows first 6 chars of API key for security)
Write-Host "API Key starts with: $($env:DD_API_KEY.Substring(0,6))..."
Write-Host "Site: $env:DD_SITE"
```

**What to expect:**
- You should see output like: `API Key starts with: abc123...` and your site URL
- **Note:** These variables only persist for the current PowerShell session

**Troubleshooting:**
- If you get a substring error, your API key might be too short or not set
- API keys are typically 32 characters long
- Double-check your site URL matches your Datadog account region

---

## 2) Run Datadog Agent with Docker

What/Why:
- Start a single Datadog Agent container that will auto-discover other containers and collect metrics/logs. APM is enabled on port 8126.

**What we're learning:** How to deploy the Datadog Agent as a Docker container to monitor your local Docker environment.

**Why this matters:** The Datadog Agent is the core component that collects metrics, logs, and traces from your infrastructure. Running it in Docker demonstrates auto-discovery capabilities - the agent will automatically find and monitor other containers without manual configuration.

**Key concepts:**
- **Auto-discovery:** Agent automatically detects running containers and starts collecting metrics
- **Docker socket mounting:** Gives the agent access to Docker metadata and container information
- **APM (Application Performance Monitoring):** Traces application requests across services
- **Log collection:** Captures container stdout/stderr logs

**Where to run:** In your PowerShell terminal, from any directory.

**Start the Datadog Agent container:**
```powershell
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

**Command explanation:**
- `-d`: Run in detached mode (background)
- `-e DD_API_KEY=$env:DD_API_KEY`: Pass your API key to the container
- `-e DD_APM_ENABLED=true`: Enable Application Performance Monitoring
- `-e DD_LOGS_ENABLED=true`: Enable log collection from containers
- `-e DD_CONTAINER_EXCLUDE="name:datadog-agent"`: Don't monitor the agent itself
- `-v /var/run/docker.sock:/var/run/docker.sock:ro`: Mount Docker socket (read-only)
- `-p 8126:8126`: Expose APM trace collection port

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

**Expected results checklist:**
- ✅ Agent container running: `docker ps` shows `datadog-agent` container in "Up" status
- ✅ No authentication errors in agent logs
- ✅ Web container accessible at http://localhost:8082 (shows NGINX welcome page)
- ✅ Container metrics visible in Datadog UI within 2-3 minutes
- ✅ Container logs flowing to Datadog Logs section within 1-2 minutes
- ✅ HTTP access logs appear in Datadog after generating traffic
- ✅ CPU and memory graphs show activity spikes during traffic generation

**Troubleshooting if data doesn't appear:**
- **Agent not connecting:** Check API key and site configuration with `docker logs datadog-agent`
- **No containers visible:** Ensure Docker socket is properly mounted (`/var/run/docker.sock`)
- **No logs appearing:** Verify `DD_LOGS_ENABLED=true` and check container exclusions
- **Network issues:** Confirm outbound access to your Datadog site (port 443 HTTPS)
- **Still no data after 10 minutes:** Restart the agent: `docker restart datadog-agent`

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

**What we're learning:** How to generate meaningful logs and understand the difference between basic monitoring (logs/metrics) and advanced Application Performance Monitoring (APM traces).

**Why this matters:** Real applications generate multiple types of telemetry data. While container metrics and logs provide infrastructure visibility, APM traces show how requests flow through your application components, helping identify performance bottlenecks and errors.

**Key concepts:**
- **Logs:** Text-based event records from applications (stdout/stderr)
- **Metrics:** Numerical measurements over time (CPU, memory, request count)
- **Traces:** Request journey across multiple services showing latency and errors
- **APM instrumentation:** Code changes needed to generate trace data

**Where to run:** Commands can be run from your PowerShell terminal.

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

**What we're learning:** How to properly clean up monitoring infrastructure to avoid resource consumption.

**Why this matters:** Leaving monitoring agents running can consume system resources and, in cloud environments, incur costs. Proper cleanup is essential for lab environments.

**Where to run:** In your PowerShell terminal.

**Clean up commands:**
```powershell
# Remove Docker containers
docker rm -f datadog-agent web

# Remove Kubernetes resources
helm uninstall datadog -n datadog
kubectl delete namespace demo
kubectl delete namespace datadog

# Verify cleanup
docker ps -a | Select-String "datadog-agent|web"
kubectl get pods --all-namespaces | Select-String "datadog|demo"
```

**Expected results:**
- ✅ Docker containers stopped and removed
- ✅ Kubernetes namespaces and pods deleted
- ✅ Helm releases uninstalled
- ✅ No Datadog-related processes consuming resources

**What happens to data in Datadog:**
- Historical metrics and logs remain in your Datadog account
- Live data stops flowing once agents are removed
- Dashboards and monitors remain configured

---

## Troubleshooting and Tips

**Common Authentication Issues:**
- **Error: "API key invalid"**
  - Solution: Verify API key is correctly set: `echo $env:DD_API_KEY`
  - Check your Datadog account: Organization Settings → API Keys
  - Ensure the key has the correct permissions

- **Error: "Unable to connect to Datadog site"**
  - Solution: Verify site URL matches your account region
  - Common sites: `us5.datadoghq.com`, `us3.datadoghq.com`, `datadoghq.eu`
  - Check firewall/proxy settings for outbound HTTPS traffic

**Docker-specific Issues:**
- **No containers visible in Infrastructure view**
  - Solution: Ensure Docker socket is properly mounted: `/var/run/docker.sock:/var/run/docker.sock:ro`
  - Verify agent container is running: `docker ps | Select-String datadog-agent`
  - Check agent logs: `docker logs datadog-agent | Select-String ERROR`

- **Agent container won't start**
  - Solution: Check Docker Desktop is running and in Linux container mode
  - Verify environment variables are set before running docker command
  - Try running with explicit values instead of environment variables

**Kubernetes-specific Issues:**
- **Pods stuck in Pending state**
  - Solution: Check node resources: `kubectl describe nodes`
  - Verify image can be pulled: `kubectl -n datadog describe pods`
  - Check for resource quotas or admission controllers

- **No Kubernetes data in Datadog**
  - Solution: Verify network connectivity: `kubectl -n datadog logs -l app=datadog`
  - Check RBAC permissions (Helm chart should handle this automatically)
  - Ensure cluster can reach your Datadog site (check egress rules)

**Data Collection Issues:**
- **Logs not appearing**
  - Solution: Confirm `DD_LOGS_ENABLED=true` or `datadog.logs.enabled=true`
  - Check log exclusion patterns: look for `DD_CONTAINER_EXCLUDE` settings
  - Verify containers are producing stdout/stderr logs

- **Metrics missing or delayed**
  - Solution: Allow 2-5 minutes for initial metric collection
  - Check agent status: `docker exec datadog-agent agent status`
  - In Kubernetes: `kubectl -n datadog exec -it <pod-name> -- agent status`

**PowerShell-specific Tips:**
- Use `Select-String` instead of `grep` for filtering command output
- Escape backticks in multi-line commands with backquotes
- Set `$ErrorActionPreference = "Stop"` to catch errors in scripts
- Use `try-catch` blocks for robust error handling in loops

**Getting Help:**
- Datadog documentation: docs.datadoghq.com
- Datadog support: Available through your account dashboard
- Community: Datadog Slack community and Stack Overflow
- Agent logs: Always check logs first for specific error messages
