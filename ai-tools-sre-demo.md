# AI Tools for SRE and Operations Demo

*A casual guide to leveraging AI for daily SRE and operational tasks*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Common AI Tools: Warp, Cursor, and Copilot](#common-ai-tools-warp-cursor-and-copilot)
3. [Shell Scripting with AI](#shell-scripting-with-ai)
4. [Kubernetes Debugging & Operations](#kubernetes-debugging--operations)
5. [Programming Scripts for Automation](#programming-scripts-for-automation)
6. [Real-World Scenarios](#real-world-scenarios)
7. [Tips & Best Practices](#tips--best-practices)
8. [Demo Script](#demo-script)

---

## Introduction

### What We'll Cover
- How AI can accelerate SRE workflows
- Building robust shell scripts quickly
- Kubernetes debugging techniques
- Automation script development
- Real operational scenarios

### Why AI for SRE?
- **Speed**: Generate complex scripts in seconds
- **Learning**: Understand unfamiliar commands and concepts
- **Debugging**: Get help troubleshooting issues
- **Best Practices**: Learn proper patterns and techniques

---

## Common AI Tools: Warp, Cursor, and Copilot

- Warp (AI-powered terminal with agents)
  - Best for: Terminal-centric workflows where you want AI to propose and execute commands, inspect output, and iterate quickly.
  - How to use: Describe the task; Warp’s agents can run commands, read/edit files, and leverage saved context (Warp Drive, MCP, Rules). You control autonomy per action (e.g., Always ask vs. Always allow) and can define YOLO/Prod profiles with allow/deny lists. Run multiple agents in parallel and step in for approvals.
  - SRE examples: Generate kubectl/terraform commands, run diagnostics from logs/stack traces, apply code diffs to shell scripts or small tools, and automate repetitive CLI flows.

- Cursor (AI IDE for code-first workflows)
  - Best for: Editing codebases, building scripts/services, and performing multi-file refactors with chat grounded in your repository.
  - How to use: Open your repo in Cursor, ask for scripts, refactors, tests, or docs; review diffs before applying. Great for turning rough shell scripts into maintainable Python/Go utilities.
  - SRE examples: Author Python/Go tooling for ops, refactor bash to Python, generate CI config, add tests around deployment/rollback logic, document runbooks inside the repo.

- GitHub Copilot (inline code completion + chat)
  - Best for: Fast, context-aware completions and small snippets across many languages; quick prompts for explanations or boilerplate.
  - How to use: Let Copilot suggest code while you type; use Copilot Chat for explanations, conversions, or generating YAML/queries.
  - SRE examples: Scaffold scripts, write PromQL, produce Kubernetes YAML, Dockerfiles, and GitHub Actions; draft PR descriptions and summarize diffs.

Tip: Use them together. Example flow: Warp to diagnose and run commands; Cursor to build/refactor a robust tool; Copilot to speed up inline edits and boilerplate.

## Shell Scripting with AI

### Quick Script Generation

**Example Prompt**: *"Create a shell script that monitors disk usage and sends alerts when it exceeds 80%"*

```bash
#!/bin/bash
# Disk usage monitoring script

THRESHOLD=80
EMAIL="admin@company.com"

check_disk_usage() {
    df -h | awk 'NR>1 {print $5 " " $6}' | while read output; do
        usage=$(echo $output | awk '{print $1}' | sed 's/%//')
        partition=$(echo $output | awk '{print $2}')
        
        if [ $usage -ge $THRESHOLD ]; then
            echo "ALERT: Disk usage on $partition is ${usage}%"
            # Send email alert (uncomment if mail is configured)
            # echo "Disk usage on $partition is ${usage}%" | mail -s "Disk Alert" $EMAIL
        fi
    done
}

check_disk_usage
```

### Log Analysis Scripts

**Example Prompt**: *"Create a script to parse nginx logs and find top 10 IP addresses by request count"*

```bash
#!/bin/bash
# Nginx log analyzer

LOG_FILE="${1:-/var/log/nginx/access.log}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "Error: Log file $LOG_FILE not found"
    exit 1
fi

echo "Top 10 IP addresses by request count:"
echo "===================================="

awk '{print $1}' "$LOG_FILE" | \
sort | \
uniq -c | \
sort -nr | \
head -10 | \
awk '{printf "%-15s %s\n", $2, $1}'
```

---

## Kubernetes Debugging & Operations

### Essential Commands (From Your Environment)

```bash
# Get all namespaces
k get ns

# Check pods in specific namespace
k get pods -n nvca-system

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward ArgoCD
k port-forward -n argocd svc/argocd-server 8080:80

# Restart deployments
k rollout restart deployment/<deployment> -n <namespace>
```

### AI-Assisted Troubleshooting Scripts

**Example Prompt**: *"Create a Kubernetes diagnostic script that checks pod health, resource usage, and recent events"*

```bash
#!/bin/bash
# Kubernetes diagnostic script

NAMESPACE="${1:-default}"
POD_NAME="${2:-}"

echo "🔍 Kubernetes Diagnostics for namespace: $NAMESPACE"
echo "=================================================="

# Function to check pod status
check_pods() {
    echo "📋 Pod Status:"
    kubectl get pods -n "$NAMESPACE" -o wide
    echo ""
}

# Function to check recent events
check_events() {
    echo "📅 Recent Events:"
    kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | tail -10
    echo ""
}

# Function to check resource usage
check_resources() {
    echo "📊 Resource Usage:"
    kubectl top pods -n "$NAMESPACE" 2>/dev/null || echo "Metrics server not available"
    echo ""
}

# Function to describe specific pod
describe_pod() {
    if [[ -n "$POD_NAME" ]]; then
        echo "🔬 Detailed Pod Analysis: $POD_NAME"
        kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
        echo ""
        
        echo "📋 Pod Logs (last 50 lines):"
        kubectl logs "$POD_NAME" -n "$NAMESPACE" --tail=50
    fi
}

# Run diagnostics
check_pods
check_events
check_resources
describe_pod
```

### Network Debugging

**Example Prompt**: *"Create a script to test network connectivity between Kubernetes services"*

```bash
#!/bin/bash
# Kubernetes network connectivity tester

SOURCE_POD="$1"
TARGET_SERVICE="$2"
TARGET_PORT="${3:-80}"
NAMESPACE="${4:-default}"

if [[ -z "$SOURCE_POD" || -z "$TARGET_SERVICE" ]]; then
    echo "Usage: $0 <source-pod> <target-service> [port] [namespace]"
    exit 1
fi

echo "🌐 Testing connectivity from $SOURCE_POD to $TARGET_SERVICE:$TARGET_PORT"
echo "======================================================================="

# Test DNS resolution
echo "🔍 DNS Resolution Test:"
kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -- nslookup "$TARGET_SERVICE"
echo ""

# Test port connectivity
echo "🔌 Port Connectivity Test:"
kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -- nc -zv "$TARGET_SERVICE" "$TARGET_PORT"
echo ""

# Test HTTP response (if applicable)
if [[ "$TARGET_PORT" == "80" || "$TARGET_PORT" == "8080" ]]; then
    echo "🌐 HTTP Response Test:"
    kubectl exec -n "$NAMESPACE" "$SOURCE_POD" -- curl -I "http://$TARGET_SERVICE:$TARGET_PORT" --max-time 5
fi
```

---

## Programming Scripts for Automation

### Python Monitoring Script

**Example Prompt**: *"Create a Python script that monitors AWS resources and sends Slack notifications"*

```python
#!/usr/bin/env python3
"""
AWS Resource Monitor with Slack Notifications
"""

import boto3
import json
import requests
from datetime import datetime, timedelta

class AWSMonitor:
    def __init__(self, slack_webhook_url):
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
        self.slack_webhook = slack_webhook_url
    
    def check_instance_health(self):
        """Check EC2 instance health and CPU usage"""
        instances = self.ec2.describe_instances()
        alerts = []
        
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                state = instance['State']['Name']
                
                if state == 'running':
                    # Check CPU usage
                    cpu_usage = self.get_cpu_usage(instance_id)
                    if cpu_usage > 80:
                        alerts.append(f"🚨 High CPU usage on {instance_id}: {cpu_usage:.1f}%")
                elif state != 'terminated':
                    alerts.append(f"⚠️ Instance {instance_id} is in state: {state}")
        
        return alerts
    
    def get_cpu_usage(self, instance_id):
        """Get average CPU usage for the last 5 minutes"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(minutes=5)
        
        response = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/EC2',
            MetricName='CPUUtilization',
            Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,
            Statistics=['Average']
        )
        
        if response['Datapoints']:
            return response['Datapoints'][0]['Average']
        return 0
    
    def send_slack_alert(self, message):
        """Send alert to Slack"""
        payload = {
            "text": f"AWS Monitor Alert: {message}",
            "username": "AWS Monitor",
            "icon_emoji": ":warning:"
        }
        
        response = requests.post(self.slack_webhook, json=payload)
        return response.status_code == 200

def main():
    # Initialize monitor (replace with your webhook URL)
    monitor = AWSMonitor("https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK")
    
    # Check for alerts
    alerts = monitor.check_instance_health()
    
    if alerts:
        for alert in alerts:
            print(alert)
            monitor.send_slack_alert(alert)
    else:
        print("✅ All systems normal")

if __name__ == "__main__":
    main()
```

### Go Service Health Checker

**Example Prompt**: *"Create a Go program that checks multiple service endpoints and creates a health dashboard"*

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

type Service struct {
    Name     string `json:"name"`
    URL      string `json:"url"`
    Status   string `json:"status"`
    Response int    `json:"response_time_ms"`
    LastCheck string `json:"last_check"`
}

type HealthChecker struct {
    services []Service
    client   *http.Client
}

func NewHealthChecker(timeout time.Duration) *HealthChecker {
    return &HealthChecker{
        client: &http.Client{Timeout: timeout},
    }
}

func (hc *HealthChecker) AddService(name, url string) {
    hc.services = append(hc.services, Service{
        Name: name,
        URL:  url,
    })
}

func (hc *HealthChecker) checkService(service *Service) {
    start := time.Now()
    
    resp, err := hc.client.Get(service.URL)
    duration := time.Since(start)
    
    service.LastCheck = time.Now().Format(time.RFC3339)
    service.Response = int(duration.Milliseconds())
    
    if err != nil {
        service.Status = "ERROR"
        return
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        service.Status = "HEALTHY"
    } else {
        service.Status = "UNHEALTHY"
    }
}

func (hc *HealthChecker) CheckAll() {
    var wg sync.WaitGroup
    
    for i := range hc.services {
        wg.Add(1)
        go func(service *Service) {
            defer wg.Done()
            hc.checkService(service)
        }(&hc.services[i])
    }
    
    wg.Wait()
}

func (hc *HealthChecker) GetStatus() []Service {
    return hc.services
}

func (hc *HealthChecker) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "timestamp": time.Now().Format(time.RFC3339),
        "services":  hc.services,
    })
}

func main() {
    checker := NewHealthChecker(5 * time.Second)
    
    // Add services to monitor
    checker.AddService("API Gateway", "https://api.example.com/health")
    checker.AddService("Database", "https://db.example.com/ping")
    checker.AddService("Cache", "https://cache.example.com/status")
    
    // Start HTTP server for dashboard
    http.Handle("/health", checker)
    
    // Periodic health checks
    go func() {
        ticker := time.NewTicker(30 * time.Second)
        for range ticker.C {
            checker.CheckAll()
            
            // Print status
            for _, service := range checker.GetStatus() {
                status := "✅"
                if service.Status != "HEALTHY" {
                    status = "❌"
                }
                fmt.Printf("%s %s: %s (%dms)\n", status, service.Name, service.Status, service.Response)
            }
            fmt.Println("---")
        }
    }()
    
    fmt.Println("Health checker started on :8080")
    fmt.Println("Dashboard: http://localhost:8080/health")
    
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}
```

---

## Real-World Scenarios

### Scenario 1: Production Incident Response

**The Problem**: Application pods are crashing in production

**AI-Assisted Approach**:
1. **Rapid Diagnostics**: Ask AI to create a comprehensive diagnostic script
2. **Log Analysis**: Generate log parsing commands to identify patterns
3. **Rollback Strategy**: Create safe rollback procedures

**Sample AI Conversation**:
```
You: "My Kubernetes pods keep crashing. Help me create a comprehensive diagnostic script."

AI: "I'll create a script that checks pod status, resource constraints, recent events, and logs..."
```

### Scenario 2: Performance Monitoring Setup

**The Problem**: Need to set up monitoring for new microservices

**AI-Assisted Approach**:
1. **Prometheus Queries**: Generate PromQL queries for key metrics
2. **Grafana Dashboards**: Create dashboard configurations
3. **Alert Rules**: Define alerting thresholds

### Scenario 3: Automated Deployment Pipeline

**The Problem**: Manual deployments are error-prone

**AI-Assisted Approach**:
1. **CI/CD Scripts**: Generate GitHub Actions or Jenkins pipelines
2. **Health Checks**: Create post-deployment verification scripts
3. **Rollback Automation**: Build automatic rollback triggers

---

## Tips & Best Practices

### Effective AI Prompting for SRE

#### ✅ Good Prompts
- **Be Specific**: "Create a bash script that monitors nginx logs for 5xx errors and sends alerts via email"
- **Include Context**: "I'm using Kubernetes 1.24 with ArgoCD, help me create a deployment health check"
- **Request Explanations**: "Explain each step of this troubleshooting process"

#### ❌ Avoid Vague Prompts
- "Fix my server"
- "Make monitoring better"
- "Help with Kubernetes"

### Security Considerations

1. **Never Share Secrets**: Redact sensitive information from prompts
2. **Review Generated Code**: Always review scripts before running in production
3. **Use Environment Variables**: For credentials and sensitive configuration

### Integration with Existing Tools

- **Terminal Integration**: Use AI directly in your terminal (like Warp AI)
- **IDE Plugins**: Leverage AI coding assistants
- **Documentation**: Generate and maintain operational runbooks

---

## Demo Script

### 5-Minute Demo Flow

1. **Introduction** (30 seconds)
   - "Today I'll show how AI accelerates SRE work"

2. **Shell Script Generation** (2 minutes)
   - Live generate a log monitoring script
   - Show how AI explains each command
   - Demonstrate rapid iteration

3. **Kubernetes Troubleshooting** (2 minutes)
   - Ask AI to help debug a pod issue
   - Generate diagnostic commands
   - Create automated health checks

4. **Automation Script** (30 seconds)
   - Quick example of monitoring script generation
   - Show how AI suggests best practices

### Interactive Elements

- **Live Coding**: Generate scripts during the demo
- **Problem-Solving**: Present a real scenario and solve it with AI
- **Q&A**: Answer questions about specific use cases

### Key Takeaways

1. **AI as a Force Multiplier**: Not replacing SRE skills, but amplifying them
2. **Learning Accelerator**: Great for understanding new tools and concepts
3. **Rapid Prototyping**: Quickly test ideas and approaches
4. **Knowledge Sharing**: Generate documentation and runbooks

---

## Resources & Next Steps

### Tools to Explore
- **Warp Terminal**: AI-powered terminal
- **GitHub Copilot**: AI pair programming
- **ChatGPT/Claude**: General AI assistance
- **Cursor**: AI-powered IDE

### Learning Path
1. Start with simple script generation
2. Move to troubleshooting assistance
3. Explore automation opportunities
4. Build AI into your daily workflows

### Community
- Share generated scripts with your team
- Build a knowledge base of AI-assisted solutions
- Contribute to open-source SRE tools

---

*Demo prepared by Rob - Feel free to adapt and extend for your specific environment!*