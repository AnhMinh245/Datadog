# 04 - CÀI ĐẶT DATADOG AGENT

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ:
- Biết cách cài đặt Agent trên nhiều nền tảng
- Configure Agent cơ bản
- Verify Agent hoạt động đúng
- Troubleshoot các vấn đề thường gặp

---

## 🚀 Chuẩn Bị

### Prerequisites

```
✅ Tài khoản Datadog (Trial 14 ngày miễn phí)
✅ API Key (lấy từ Datadog UI)
✅ Quyền sudo/admin trên server
✅ Internet connection (HTTPS outbound)
```

### Lấy API Key

```
1. Login vào https://app.datadoghq.com
2. Navigate to: Organization Settings → API Keys
3. Copy API Key (hoặc tạo mới)

Example: 1234567890abcdef1234567890abcdef
```

### Chọn Site Đúng

```
US1 (Default): app.datadoghq.com
US3: us3.datadoghq.com
US5: us5.datadoghq.com
EU1: app.datadoghq.eu

→ Check URL khi bạn login
→ Dùng đúng site khi cài Agent!
```

---

## 💻 Cài Đặt Trên Linux

### Ubuntu / Debian

**One-line Installation:**

```bash
DD_AGENT_MAJOR_VERSION=7 \
DD_API_KEY=<YOUR_API_KEY> \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

**Step-by-step:**

```bash
# 1. Download and run install script
export DD_API_KEY=<YOUR_API_KEY>
export DD_SITE="datadoghq.com"

curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh \
  -o /tmp/install_datadog.sh

bash /tmp/install_datadog.sh

# 2. Verify installation
sudo datadog-agent status

# 3. Start Agent
sudo systemctl start datadog-agent

# 4. Enable auto-start
sudo systemctl enable datadog-agent
```

### CentOS / RHEL / Amazon Linux

```bash
DD_AGENT_MAJOR_VERSION=7 \
DD_API_KEY=<YOUR_API_KEY> \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"

# Start and enable
sudo systemctl start datadog-agent
sudo systemctl enable datadog-agent
```

### Manual Installation (APT)

```bash
# 1. Setup APT repository
sudo apt-get update
sudo apt-get install -y apt-transport-https curl gnupg

# 2. Add Datadog GPG key
sudo sh -c "echo 'deb [signed-by=/usr/share/keyrings/datadog-archive-keyring.gpg] \
https://apt.datadoghq.com/ stable 7' > /etc/apt/sources.list.d/datadog.list"

curl https://keys.datadoghq.com/DATADOG_APT_KEY_CURRENT.public \
  | sudo gpg --dearmor -o /usr/share/keyrings/datadog-archive-keyring.gpg

# 3. Install Agent
sudo apt-get update
sudo apt-get install -y datadog-agent

# 4. Configure API key
sudo sh -c "sed 's/api_key:.*/api_key: <YOUR_API_KEY>/' \
/etc/datadog-agent/datadog.yaml.example > /etc/datadog-agent/datadog.yaml"

# 5. Set site
sudo sh -c "echo 'site: datadoghq.com' >> /etc/datadog-agent/datadog.yaml"

# 6. Start Agent
sudo systemctl restart datadog-agent
```

---

## 🪟 Cài Đặt Trên Windows

### GUI Installation

```powershell
# 1. Download installer
# https://s3.amazonaws.com/ddagent-windows-stable/datadog-agent-7-latest.amd64.msi

# 2. Run installer (PowerShell as Administrator)
Start-Process msiexec.exe -Wait -ArgumentList '/qn /i datadog-agent-7-latest.amd64.msi APIKEY="<YOUR_API_KEY>" SITE="datadoghq.com"'

# 3. Verify
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

### Command Line Installation

```powershell
# PowerShell (Run as Administrator)

# Download
$agentUrl = "https://s3.amazonaws.com/ddagent-windows-stable/datadog-agent-7-latest.amd64.msi"
$installerPath = "$env:TEMP\datadog-agent.msi"
Invoke-WebRequest -Uri $agentUrl -OutFile $installerPath

# Install
$apiKey = "<YOUR_API_KEY>"
$site = "datadoghq.com"

msiexec /qn /i $installerPath APIKEY="$apiKey" SITE="$site"

# Check status
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

### Config File Location (Windows)

```
C:\ProgramData\Datadog\datadog.yaml
C:\ProgramData\Datadog\conf.d\
```

---

## 🐳 Cài Đặt Trên Docker

### Single Container

```bash
docker run -d --name dd-agent \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -e DD_API_KEY=<YOUR_API_KEY> \
  -e DD_SITE="datadoghq.com" \
  -e DD_LOGS_ENABLED=true \
  -e DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true \
  -e DD_APM_ENABLED=true \
  -e DD_APM_NON_LOCAL_TRAFFIC=true \
  datadog/agent:latest
```

**Explanation:**
```
-v /var/run/docker.sock     # Monitor Docker
-v /proc/                   # System metrics
-v /sys/fs/cgroup/          # Container metrics
-e DD_LOGS_ENABLED          # Enable log collection
-e DD_APM_ENABLED           # Enable APM
-e DD_APM_NON_LOCAL_TRAFFIC # Accept traces from other containers
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3'
services:
  datadog:
    image: datadog/agent:latest
    container_name: dd-agent
    environment:
      - DD_API_KEY=${DD_API_KEY}
      - DD_SITE=datadoghq.com
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
      - DD_APM_ENABLED=true
      - DD_APM_NON_LOCAL_TRAFFIC=true
      - DD_PROCESS_AGENT_ENABLED=true
      - DD_DOCKER_LABELS_AS_TAGS={"my.custom.label":"custom_tag"}
      - DD_TAGS='env:production service:my-app'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
    ports:
      - "8126:8126/tcp"  # APM
      - "8125:8125/udp"  # DogStatsD
```

**Start:**
```bash
# Create .env file
echo "DD_API_KEY=<YOUR_API_KEY>" > .env

# Start
docker-compose up -d

# Check logs
docker-compose logs -f datadog

# Check status
docker exec -it dd-agent agent status
```

---

## ☸️ Cài Đặt Trên Kubernetes

### Helm Installation (Recommended)

```bash
# 1. Add Datadog Helm repo
helm repo add datadog https://helm.datadoghq.com
helm repo update

# 2. Create values file
cat <<EOF > datadog-values.yaml
datadog:
  apiKey: <YOUR_API_KEY>
  site: datadoghq.com
  logs:
    enabled: true
    containerCollectAll: true
  apm:
    portEnabled: true
  processAgent:
    enabled: true
  systemProbe:
    enabled: true
  tags:
    - env:production
    - cluster:my-cluster

clusterAgent:
  enabled: true
  metricsProvider:
    enabled: true  # For HPA with custom metrics

agents:
  useHostNetwork: true
EOF

# 3. Install
helm install datadog-agent datadog/datadog \
  -f datadog-values.yaml \
  --namespace datadog \
  --create-namespace

# 4. Verify
kubectl get pods -n datadog
kubectl exec -it <POD_NAME> -n datadog -- agent status
```

### DaemonSet (Manual)

```yaml
# datadog-agent.yaml
apiVersion: v1
kind: Secret
metadata:
  name: datadog-secret
  namespace: datadog
type: Opaque
stringData:
  api-key: <YOUR_API_KEY>
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
  namespace: datadog
spec:
  selector:
    matchLabels:
      app: datadog-agent
  template:
    metadata:
      labels:
        app: datadog-agent
      name: datadog-agent
    spec:
      serviceAccountName: datadog-agent
      containers:
      - name: datadog-agent
        image: datadog/agent:latest
        env:
        - name: DD_API_KEY
          valueFrom:
            secretKeyRef:
              name: datadog-secret
              key: api-key
        - name: DD_SITE
          value: "datadoghq.com"
        - name: DD_LOGS_ENABLED
          value: "true"
        - name: DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL
          value: "true"
        - name: DD_APM_ENABLED
          value: "true"
        - name: DD_KUBERNETES_KUBELET_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: dockersocket
          mountPath: /var/run/docker.sock
        - name: procdir
          mountPath: /host/proc
          readOnly: true
        - name: cgroups
          mountPath: /host/sys/fs/cgroup
          readOnly: true
      volumes:
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock
      - name: procdir
        hostPath:
          path: /proc
      - name: cgroups
        hostPath:
          path: /sys/fs/cgroup
```

---

## ☁️ Cài Đặt Trên Cloud Platforms

### AWS EC2

**Using AWS Systems Manager:**

```bash
# 1. Create SSM parameter for API key
aws ssm put-parameter \
  --name "/datadog/api_key" \
  --value "<YOUR_API_KEY>" \
  --type "SecureString"

# 2. User data script for EC2
#!/bin/bash
DD_API_KEY=$(aws ssm get-parameter --name /datadog/api_key --with-decryption --query 'Parameter.Value' --output text)
DD_AGENT_MAJOR_VERSION=7 \
DD_API_KEY=$DD_API_KEY \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

**Auto-tagging:**

```yaml
# /etc/datadog-agent/datadog.yaml
collect_ec2_tags: true

tags:
  - team:backend
  - project:ecommerce
```

### AWS ECS (Fargate)

```json
{
  "family": "datadog-agent-task",
  "containerDefinitions": [
    {
      "name": "datadog-agent",
      "image": "datadog/agent:latest",
      "essential": true,
      "environment": [
        {"name": "DD_API_KEY", "value": "<YOUR_API_KEY>"},
        {"name": "DD_SITE", "value": "datadoghq.com"},
        {"name": "ECS_FARGATE", "value": "true"},
        {"name": "DD_APM_ENABLED", "value": "true"}
      ],
      "portMappings": [
        {"containerPort": 8126, "protocol": "tcp"}
      ]
    }
  ]
}
```

### Google Cloud (GCE)

```bash
# Startup script
#!/bin/bash
DD_API_KEY=$(gcloud secrets versions access latest --secret="datadog-api-key")
DD_AGENT_MAJOR_VERSION=7 \
DD_API_KEY=$DD_API_KEY \
DD_SITE="datadoghq.com" \
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"

# Enable GCP metadata collection
echo "collect_gce_tags: true" >> /etc/datadog-agent/datadog.yaml
```

### Azure VM

```powershell
# Custom Script Extension
$apiKey = (Get-AzKeyVaultSecret -VaultName "mykeyvault" -Name "datadog-api-key").SecretValueText

DD_AGENT_MAJOR_VERSION=7 `
DD_API_KEY=$apiKey `
DD_SITE="datadoghq.com" `
bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

---

## ⚙️ Configuration Cơ Bản

### Main Config File

```yaml
# /etc/datadog-agent/datadog.yaml

## REQUIRED
api_key: <YOUR_API_KEY>
site: datadoghq.com

## TAGS
tags:
  - env:production
  - service:my-app
  - team:backend
  - version:1.0.0

## HOSTNAME
hostname: my-server-01
# hostname_fqdn: true  # Use FQDN instead

## LOGS
logs_enabled: true
logs_config:
  container_collect_all: true
  processing_rules:
    - type: exclude_at_match
      name: exclude_debug
      pattern: DEBUG

## APM
apm_config:
  enabled: true
  apm_non_local_traffic: true  # Accept from other hosts

## PROCESS MONITORING
process_config:
  enabled: "true"
  
## DOGSTATSD
dogstatsd_port: 8125
dogstatsd_non_local_traffic: true

## PROXY (if needed)
# proxy:
#   https: http://proxy.company.com:3128
#   no_proxy:
#     - 169.254.169.254
```

### Common Configurations

**1. Custom Tags:**
```yaml
tags:
  - env:production
  - datacenter:us-east-1
  - role:web
  - team:payments
```

**2. Hostname Override:**
```yaml
hostname: custom-hostname
# Useful when multiple containers/processes on same host
```

**3. Log Collection:**
```yaml
logs_enabled: true
logs_config:
  container_collect_all: true
  open_files_limit: 100
```

**4. Enable APM:**
```yaml
apm_config:
  enabled: true
  apm_non_local_traffic: true
  analyzed_spans:
    my-service|endpoint: 1  # Sample 100%
```

---

## ✅ Verify Installation

### Check Agent Status

```bash
# Linux
sudo datadog-agent status

# Windows
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status

# Docker
docker exec dd-agent agent status

# Kubernetes
kubectl exec -it <POD_NAME> -n datadog -- agent status
```

**Expected Output:**
```
===============
Agent (v7.49.1)
===============

  Status date: 2026-01-01 10:30:15.123 UTC
  Agent start: 2026-01-01 10:00:00.000 UTC
  Pid: 12345
  Go Version: go1.21.0
  Python Version: 3.11.0
  Build arch: amd64
  Agent flavor: agent
  Log Level: info

  Paths
  =====
    Config File: /etc/datadog-agent/datadog.yaml
    conf.d: /etc/datadog-agent/conf.d
    checks.d: /etc/datadog-agent/checks.d

  Clocks
  ======
    NTP offset: 0.0012s
    System UTC time: 2026-01-01 10:30:15.123 UTC

  Host Info
  =========
    Hostname: my-server-01
    OS: linux
    Platform: ubuntu
    Kernel Version: 5.15.0
    Processor: Intel(R) Xeon(R) CPU @ 2.50GHz
    ...

  Collector
  =========
    Running Checks
    ==============
      cpu (1.0.0)
      ---------------
        Instance ID: cpu [OK]
        Total Runs: 120
        Metric Samples: 7, Total: 840
        Events: 0, Total: 0
        Service Checks: 0, Total: 0
      
      disk (1.0.0)
      ------------
        Instance ID: disk:abc123 [OK]
        ...
        
  Logs Agent
  ==========
    Reliable: true
    Status: Running
    Inputs: ...
```

### Send Test Metric

```bash
# Using datadog-agent command
echo -n "custom.metric:1|g|#env:test" | \
  nc -u -w1 localhost 8125

# Using API
curl -X POST "https://api.datadoghq.com/api/v1/series" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: <YOUR_API_KEY>" \
  -d '{
    "series": [{
      "metric": "test.metric",
      "points": [['"$(date +%s)"', 1]],
      "type": "gauge",
      "host": "test-host",
      "tags": ["env:test"]
    }]
  }'
```

### Check in UI

```
1. Navigate to: Infrastructure → Infrastructure List
   → Should see your host

2. Navigate to: Metrics → Explorer
   → Search for "system.cpu.user"
   → Filter by your hostname

3. Navigate to: Dashboards → System - Overview
   → Select your host
```

---

## 🔧 Troubleshooting

### Agent Not Starting

**Check logs:**
```bash
# Linux
sudo tail -f /var/log/datadog/agent.log

# Windows
Get-Content "C:\ProgramData\Datadog\logs\agent.log" -Tail 50 -Wait

# Docker
docker logs dd-agent -f
```

**Common issues:**
```
❌ Invalid API key
   → Check datadog.yaml, verify key in UI

❌ Wrong site
   → Check site setting matches your login URL

❌ Network/Firewall
   → Test: curl https://api.datadoghq.com
   → Check firewall allows port 443 outbound

❌ Permissions
   → Agent needs read access to /proc, /sys
   → Check file permissions
```

### Agent Running But No Data

```bash
# 1. Check connectivity
sudo datadog-agent diagnose

# 2. Flare (send diagnostics to support)
sudo datadog-agent flare

# 3. Check specific integration
sudo datadog-agent check cpu

# 4. Verify config
sudo datadog-agent config
```

### High CPU/Memory Usage

```yaml
# Tune config in datadog.yaml

# Reduce check frequency
min_collection_interval: 30  # Default: 15 seconds

# Reduce log processing
logs_config:
  open_files_limit: 50  # Default: 100

# Disable unused features
process_config:
  enabled: "false"
```

---

## 📝 Tóm Tắt

### Installation Checklist

```
✅ Choose right platform (Linux/Windows/Docker/K8s)
✅ Get API key from Datadog UI
✅ Set correct site (US1/EU1/etc)
✅ Run installation command
✅ Configure datadog.yaml
✅ Add tags
✅ Enable features (logs, APM)
✅ Start agent service
✅ Verify with: datadog-agent status
✅ Check data in UI
```

### Key Commands

```bash
# Status
datadog-agent status

# Start/Stop/Restart
systemctl start datadog-agent
systemctl stop datadog-agent
systemctl restart datadog-agent

# Check specific integration
datadog-agent check <CHECK_NAME>

# View config
datadog-agent config

# Diagnostics
datadog-agent diagnose

# Send flare (support bundle)
datadog-agent flare <CASE_ID>
```

### Config File Locations

```
Linux:    /etc/datadog-agent/datadog.yaml
Windows:  C:\ProgramData\Datadog\datadog.yaml
Docker:   Environment variables
K8s:      ConfigMap or Helm values
```

---

## 🎯 Bài Tập

### Exercise 1: Basic Installation

```
1. Tạo tài khoản Datadog trial
2. Cài Agent trên local VM/machine
3. Verify host xuất hiện trong UI
4. Take screenshot của Infrastructure List
```

### Exercise 2: Configuration

```
1. Add custom tags:
   - env:lab
   - student:your-name
   
2. Enable logs collection

3. Verify tags trong UI

4. Tìm metric: system.cpu.user
   với tags của bạn
```

### Exercise 3: Docker Lab

```
1. Run Datadog Agent container

2. Run example app container:
   docker run -d --name web nginx

3. Verify Agent monitors the nginx container

4. Check logs của nginx trong Datadog UI
```

### Exercise 4: Troubleshooting

```
Scenario: Agent installed but no data

Steps:
1. Check agent status
2. Check logs
3. Test connectivity
4. Verify API key
5. Check tags

Document your findings
```

---

## ➡️ Bước Tiếp Theo

Bạn đã cài đặt thành công Agent!

**Bài tiếp theo**: [05 - Dashboard và Visualization](05-DASHBOARD.md)

Trong bài tiếp theo, chúng ta sẽ học cách tạo dashboard để visualize metrics.

---

**📌 Ghi Chú Của Bạn**
```
(Viết command đã dùng, lỗi gặp phải, solutions, etc.)








```

