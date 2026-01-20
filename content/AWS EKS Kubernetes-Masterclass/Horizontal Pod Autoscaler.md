---
longform:
  format: single
  title: Horizontal Pod Autoscaler
title: Horizontal Pod Autoscaler
---
# EKS Horizontal Pod Autoscaling (HPA)

## What is HPA?
Horizontal Pod Autoscaling automatically adjusts the number of pods in a deployment based on observed CPU/memory utilization or custom metrics.

## 1. Install Metrics Server
HPA needs metrics from the Metrics Server to make scaling decisions.

```bash
# Check if Metrics Server is installed
kubectl -n kube-system get deployment/metrics-server

# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

# Verify installation
kubectl get deployment metrics-server -n kube-system
```

## 2. Deploy Sample Application

Create manifest file `hpa-demo.yaml`:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo-deployment
  labels:
    app: hpa-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-nginx
  template:
    metadata:
      labels:
        app: hpa-nginx
    spec:
      containers:
      - name: hpa-nginx
        image: dash04/oldapp:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "500Mi"
            cpu: "200m"          
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo-service-nginx
  labels:
    app: hpa-nginx
spec:
  type: NodePort
  selector:
    app: hpa-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31231
```

Apply the configuration:

```bash
# Deploy Nginx application with Service
kubectl apply -f hpa-demo.yaml

# Verify deployment
kubectl get pod,svc,deploy

# Access application (if in public subnet)
# Get node IP: kubectl get nodes -o wide
# Access: http://<Worker-Node-Public-IP>:31231
```

## 3. Create HPA
Create an autoscaler targeting 50% CPU utilization, scaling between 1-10 pods:

```bash
# Create HPA
kubectl autoscale deployment hpa-demo-deployment --cpu-percent=50 --min=1 --max=10

# Check HPA
kubectl describe hpa/hpa-demo-deployment 
kubectl get hpa
```

**How it works:**
- When CPU usage > 50% → increases pods (up to 10)
- When CPU usage < 50% → decreases pods (down to 1)

## 4. Generate Load to Trigger Scaling
```bash
# Run load test using Apache Bench
kubectl run apache-bench \
  --image=httpd \
  --restart=Never \
  --rm -i --tty \
  -- ab -n 1000000 -c 1000 http://hpa-demo-service-nginx.default.svc.cluster.local/

# Monitor HPA
kubectl get hpa hpa-demo-deployment -w

# Check pods scaling up
kubectl get pods

# Describe HPA for detailed metrics
kubectl describe hpa/hpa-demo-deployment
```

**Command Explanation:**
- `kubectl run apache-bench`: Creates a temporary pod
- `--image=httpd`: Uses Apache HTTPD image (contains `ab` tool)
- `--rm`: Remove pod after completion
- `-i --tty`: Interactive terminal
- `ab`: Apache Bench tool
- `-n 1000000`: 1 million requests
- `-c 1000`: 1000 concurrent connections
- `http://...`: Target service URL

## 5. Scale Down (Cooldown)
After load stops, HPA waits 5 minutes (default) before scaling down to prevent rapid fluctuations.

## 6. Clean Up
```bash
# Delete HPA
kubectl delete hpa hpa-demo-deployment

# Delete application deployment 
kubectl delete deploy hpa-demo-deployment
```

## HPA Behavior Configuration (v1.18+)
You can customize scaling behavior using YAML:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
      - type: Percent
        value: 100  # Remove all excess pods at once
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Double pods
        periodSeconds: 15
      - type: Pods
        value: 4    # Or add 4 pods at minimum
        periodSeconds: 15
      selectPolicy: Max  # Use most aggressive scaling option
```

## Key Points:
- **Metrics Server** must be installed for CPU/memory-based scaling
- **Default cooldown**: 5 minutes for scale-down
- **Scaling policies** customizable in v1.18+
- HPA uses **averaged metrics** across all pods
- Supports **custom metrics** and **multiple metrics**