Step 1: Create Namespace for Monitoring

```
kubectl create namespace monitoring
```
Step 2: Add kube-prometheus-stack Helm Chart Repository

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
Step 3: Install Prometheus using Helm

```
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```
Step 4: Update Helm Repositories

```
helm repo update
```
Step 5: Verify Prometheus Installation

```
kubectl get pods -n monitoring
```


![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/7cce65cd-5190-4d5f-a6ec-2cd152622534)

![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/3dfdc449-0070-4a19-b382-614036908870)


Step 6: Create Nginx Deployment

```
kubectl create deployment nginx --image=nginx --port 80 -n web
```
Step 7: Expose Nginx Deployment as a Service

```
kubectl expose deployment nginx --port 80 -n web
```
Step 8: Verify Nginx Service Creation

```
kubectl get svc -n web
```

![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/6aab3c21-ff43-4243-8fe3-fd5bbecb4715)


![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/6c5f989e-eb40-4f5e-9636-f4006a565477)

Create a Service Monitor YAML file. For example:nginx-service-monitor.yaml
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: nginx
  endpoints:
  - port: web
```
Apply the Service Monitor YAML file:
```
kubectl apply -f nginx-service-monitor.yaml
```
![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/40b9d7a7-6256-4a96-9ebb-97e44b9d67f6)


Step 9: Install KEDA with Helm:

Add the KEDA Helm repository:
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
```
Install KEDA:
```
helm install keda kedacore/keda -n keda
```
![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/ef623960-eb83-4c2e-a9ac-372acff5fd9d)


Step 10: Create ScaledObject for Nginx Autoscaling:

Create a ScaledObject CRD  file. For example:nginxScaledObject.yaml 
```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-scaler
  namespace: web
spec:
  scaleTargetRef:
    name: nginx
  pollingInterval: 30
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      query: "sum(rate(http_requests_total{job='nginx'}[1m]))"
      threshold: "10"
      panicThreshold: "100"
```
Apply the ScaledObject YAML file:
```
kubectl apply -f nginxScaledObject.yaml
```

Get the ClusterIP and port of your Nginx service:
```
kubectl get svc -n web
```
Use curl to generate HTTP traffic to your Nginx service:
```
curl http://<nginx-service-ip>:<nginx-service-port>
```
```
curl http://10.102.253.44:80
```
![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/12b6793c-e55f-42b7-998c-ee1a698119e8)

Monitor the number of pods in the web namespace:
```
kubectl get pods -n web
```

Observe if the number of pods increases beyond the specified maximum replica count in your ScaledObject YAML file when there is more traffic.

![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/17b32ed4-8560-48ea-a729-0ff9283f6682)





Once traffic decreases, observe if the number of pods scales down to the specified minimum replica count.
![image](https://github.com/vijaybiradar/KEDA-/assets/38376802/47f5acba-86a6-47c6-8398-24f2678953d5)



Step 11: Add Karpenter Helm Repository

```
helm repo add karpenter https://awslabs.github.io/karpenter/charts
helm repo update
```
Step 12: Install Karpenter with Helm

```
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --set clusterName=<your-cluster-name> \
  --set cloudProvider.name=<your-cloud-provider> \
  --set metricsProvider.prometheus.enabled=true \
  --set metricsProvider.prometheus.endpoint="http://prometheus-kube-prometheus-stack-prometheus.monitoring.svc:9090"
```
Replace <your-cluster-name> with the name of your Kubernetes cluster and <your-cloud-provider> with your cloud provider's name.

Step 13: Create a Karpenter Provisioner YAML file

Create a YAML file (e.g., karpenter-provisioner.yaml) to define the Karpenter Provisioner configuration:

```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  provider:
    subnetSelector:
      kubernetes.io/cluster/<cluster-name>: '*'
    securityGroupSelector:
      kubernetes.io/cluster/<cluster-name>: '*'
    instanceProfile: <launch-profile-name>
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
```
Replace <cluster-name> with the name of your Kubernetes cluster and <launch-profile-name> with the name of the launch profile to be used for new nodes.

Step 4: Apply the Karpenter Provisioner YAML file

Apply the Karpenter Provisioner configuration to your Kubernetes cluster:

```
kubectl apply -f karpenter-provisioner.yaml -n karpenter
```
Step 5: Monitor Karpenter

Monitor the Karpenter pods to ensure they are running properly:

```
kubectl get pods -n karpenter
```



