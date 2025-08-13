# Splunk Lab - Common Issues and Solutions

This document captures the real-world issues encountered during the Splunk lab setup and their solutions.

## Issue 1: SCK Pods in CreateContainerConfigError

### Symptoms
```
kubectl get pods -n kube-system | findstr sck
sck-splunk-kubernetes-logging-whc2q                  0/1     CreateContainerConfigError   0             4m17s
sck-splunk-kubernetes-metrics-agg-7c5f7c7f4c-gfgbt   0/1     CreateContainerConfigError   0             5m52s
```

### Root Cause
The Helm installation didn't properly create secrets with HEC token data because the `$hecToken` PowerShell variable was not set or was empty.

### Diagnosis Steps
1. **Check pod details**:
   ```powershell
   kubectl describe pod sck-splunk-kubernetes-logging-whc2q -n kube-system
   ```

2. **Look for the error message**:
   ```
   Events:
   Warning  Failed  kubectl  Error: couldn't find key splunk_hec_token in Secret kube-system/splunk-kubernetes-logging
   ```

3. **Check secret contents**:
   ```powershell
   kubectl get secrets -n kube-system | findstr splunk
   kubectl describe secret splunk-kubernetes-logging -n kube-system
   ```

4. **Verify secret has no data**:
   ```
   Type:  Opaque
   Data
   ====
   (empty - 0 entries)
   ```

5. **Check Helm values**:
   ```powershell
   helm get values sck -n kube-system
   ```
   Shows: `token: ""` (empty)

### Solution
1. **Set the token variable**:
   ```powershell
   $hecToken = "00000000-0000-0000-0000-000000000000"
   echo "Token set to: $hecToken"
   ```

2. **Upgrade Helm release with correct token**:
   ```powershell
   helm upgrade sck splunk/splunk-connect-for-kubernetes `
     --namespace kube-system `
     --set global.splunk.hec.host=host.docker.internal `
     --set global.splunk.hec.port=8088 `
     --set global.splunk.hec.protocol=http `
     --set global.splunk.hec.token=$hecToken `
     --set logging.enabled=true `
     --set metrics.enabled=true `
     --set objects.enabled=false
   ```

3. **Verify secret now contains data**:
   ```powershell
   kubectl describe secret splunk-kubernetes-logging -n kube-system
   ```
   Should show: `splunk_hec_token:  36 bytes`

---

## Issue 2: Metrics Pods in CrashLoopBackOff

### Symptoms
```
sck-splunk-kubernetes-metrics-agg-7fc8bd7776-lrvnv   0/1     CrashLoopBackOff   2 (22s ago)   9m8s
sck-splunk-kubernetes-metrics-fzbwk                  0/1     CrashLoopBackOff   2 (10s ago)   9m7s
```

### Root Cause
Metrics components require protocol setting in the `global.splunk.hec.*` section, not just `splunk.hec.*`.

### Diagnosis Steps
1. **Check pod logs**:
   ```powershell
   kubectl logs sck-splunk-kubernetes-metrics-fzbwk -n kube-system --tail=20
   ```

2. **Look for config error**:
   ```
   [error]: config error file="/fluentd/etc/fluent.conf" error_class=Fluent::ConfigError error="valid options are http,https but got "
   ```

### Solution
Use `global.splunk.hec.*` for all settings to ensure they apply to all components:

```powershell
helm upgrade sck splunk/splunk-connect-for-kubernetes `
  --namespace kube-system `
  --set global.splunk.hec.host=host.docker.internal `
  --set global.splunk.hec.port=8088 `
  --set global.splunk.hec.protocol=http `
  --set global.splunk.hec.token=$hecToken `
  --set logging.enabled=true `
  --set metrics.enabled=true `
  --set objects.enabled=false
```

---

## Issue 3: Docker Log Driver SSL/TLS Errors

### Symptoms
```
docker: Error response from daemon: failed to create task for container: failed to initialize logging driver: Options "http://host.docker.internal:8088/services/collector/event/1.0": EOF
```

### Root Cause
Mismatch between HEC SSL configuration and Docker log driver URL protocol.

### Solution Options

**Option 1 - Use HTTPS (recommended for matching Splunk defaults)**:
```powershell
docker run -d --name web `
  --log-driver splunk `
  --log-opt splunk-token=00000000-0000-0000-0000-000000000000 `
  --log-opt splunk-url=https://host.docker.internal:8088 `
  --log-opt splunk-insecureskipverify=true `
  --log-opt splunk-format=raw `
  -p 8081:80 nginx
```

**Option 2 - Disable HEC SSL in Splunk**:
1. Add to `.env`: `SPLUNK_HEC_SSL=false`
2. Recreate Splunk: `docker compose down -v; docker compose up -d`
3. Use `http://` in log driver URL

---

## Prevention Checklist

### Before Starting SCK Installation
- [ ] Verify `$hecToken` variable is set: `echo $hecToken`
- [ ] Confirm Splunk HEC is healthy: `curl -k https://localhost:8088/services/collector/health`
- [ ] Check Docker can resolve host.docker.internal: `docker run --rm alpine nslookup host.docker.internal`

### During Installation
- [ ] Use `global.splunk.hec.*` settings for all parameters
- [ ] Include `--create-namespace` flag
- [ ] Verify Helm values after install: `helm get values sck -n kube-system`

### After Installation
- [ ] Check all pods are Running: `kubectl get pods -n kube-system | findstr sck`
- [ ] Verify secrets contain data: `kubectl describe secret splunk-kubernetes-logging -n kube-system`
- [ ] Test with demo workload and search in Splunk

---

## Quick Recovery Commands

### Reset SCK Installation
```powershell
# Remove failed installation
helm uninstall sck -n kube-system

# Set token and reinstall
$hecToken = "00000000-0000-0000-0000-000000000000"
helm upgrade --install sck splunk/splunk-connect-for-kubernetes `
  --namespace kube-system `
  --create-namespace `
  --set global.splunk.hec.host=host.docker.internal `
  --set global.splunk.hec.port=8088 `
  --set global.splunk.hec.protocol=http `
  --set global.splunk.hec.token=$hecToken `
  --set logging.enabled=true `
  --set metrics.enabled=true `
  --set objects.enabled=false
```

### Check Pod Status and Logs
```powershell
# Quick status check
kubectl get pods -n kube-system | findstr sck

# Get logs from failing pods
kubectl logs -n kube-system -l app=splunk-kubernetes-logging --tail=50
kubectl logs -n kube-system -l app=splunk-kubernetes-metrics --tail=50

# Check secret contents
kubectl get secrets -n kube-system | findstr splunk
kubectl describe secret splunk-kubernetes-logging -n kube-system
```

### Verify End-to-End Flow
```powershell
# Create test workload
kubectl create namespace demo
kubectl -n demo create deploy hello --image=nginx --replicas=2

# Search in Splunk (wait 2-3 minutes for data)
# index=main kubernetes.namespace_name=demo
# index=main kubernetes.pod_name=*hello*
```

## 4. HEC Not Enabled Issue

**Problem**: SCK pods running but logs not reaching Splunk

**Symptoms**:
- SCK logging pod shows "pattern not matched" warnings
- Connectivity tests to HEC endpoint fail
- curl to `localhost:8088/services/collector/health` returns "Empty reply from server"

**Root Cause**: HTTP Event Collector (HEC) is disabled by default in Splunk

**Diagnosis**:
```powershell
# Check HEC configuration
docker exec splunk cat /opt/splunk/etc/apps/splunk_httpinput/default/inputs.conf

# If you see disabled=1, HEC is not enabled
```

**Solution**: Enable HEC in Splunk Web UI:

1. **Open Splunk Web**: http://localhost:8000 (admin/yourpassword)
2. **Navigate to**: Settings → Data Inputs → HTTP Event Collector
3. **Global Settings**: Click "Global Settings" button
4. **Enable HEC**: Check "All Tokens" → Enable 
5. **SSL Settings**: Set SSL to "Optional" or "Disabled" for testing
6. **Save**: Click "Save" button
7. **Restart if needed**:
   ```powershell
   docker restart splunk
   ```

**Verification**:
```powershell
# Test HEC health endpoint after enabling
Invoke-WebRequest -Uri "http://localhost:8088/services/collector/health" -UseBasicParsing

# Should return HTTP 200 with JSON response
```

**Alternative CLI Method** (if Web UI is not accessible):
```powershell
# Enable HEC via CLI (password is in container environment: Changeme123!)
docker exec splunk sudo -u splunk /opt/splunk/bin/splunk http-event-collector enable -uri https://localhost:8089 -auth admin:Changeme123!

# Restart Splunk to apply changes
docker restart splunk
```

**Update SCK Configuration**: After enabling HEC, update SCK to use HTTPS:
```powershell
$hecToken = "00000000-0000-0000-0000-000000000000"
helm upgrade sck splunk/splunk-connect-for-kubernetes `
  --namespace kube-system `
  --set global.splunk.hec.host=host.docker.internal `
  --set global.splunk.hec.port=8088 `
  --set global.splunk.hec.protocol=https `
  --set global.splunk.hec.insecureSSL=true `
  --set global.splunk.hec.token=$hecToken `
  --set logging.enabled=true `
  --set metrics.enabled=true `
  --set objects.enabled=false
```

**Final Verification**:
```powershell
# Check all SCK pods are running
kubectl get pods -n kube-system | findstr sck

# Test connectivity from SCK pod to Splunk HEC
kubectl exec $(kubectl get pods -n kube-system -l app=splunk-kubernetes-logging -o jsonpath='{.items[0].metadata.name}') -n kube-system -- curl -k https://host.docker.internal:8088/services/collector/health

# Generate test logs and search in Splunk after 2-3 minutes:
kubectl rollout restart deployment hello -n demo
# Search: index=main kubernetes.namespace_name=demo
```
