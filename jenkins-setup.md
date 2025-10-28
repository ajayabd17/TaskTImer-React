# Jenkins CI/CD Pipeline Configuration

## Prerequisites

### 1. Jenkins Plugins Required
Install these plugins in Jenkins:
- Docker Pipeline
- Kubernetes CLI
- Kubernetes Plugin
- Git Plugin
- Credentials Plugin

### 2. Credentials Setup
Create these credentials in Jenkins:

#### Docker Registry Credentials
- **ID**: `docker-registry-credentials`
- **Type**: Username with password
- **Username**: Your Docker registry username
- **Password**: Your Docker registry password/token

#### Kubernetes Config
- **ID**: `kubeconfig`
- **Type**: Secret file
- **File**: Upload your `~/.kube/config` file

## Jenkins Job Setup

1. **Create the pipeline job**
   - From the Jenkins dashboard choose **New Item**.
   - Enter the name `timer-app-deployment`, select **Pipeline**, click **OK**.
2. **Configure source control**
   - Under **Pipeline** set **Definition** to `Pipeline script from SCM`.
   - Choose **Git**, set the repository URL to `https://github.com/ajayabd17/TaskTImer-React.git`.
   - Use the default credentials (none required for public repo), set **Branch Specifier** to `*/main`, and **Script Path** to `Jenkinsfile`.
3. **Add build triggers**
   - Check **GitHub hook trigger for GITScm polling** to support webhooks.
   - Optionally enable **Poll SCM** with `H/5 * * * *` to fall back to polling.
4. **Save** the job configuration.
5. **Run the first build**
   - Open the job and click **Build Now**.
   - Watch the console log; the pipeline will build the image, deploy to the idle colour, smoke-test, switch traffic, and verify the service.
6. **Verify the deployment**
   - `kubectl get svc -n timer-app` — confirm the `track` selector matches the colour advertised in Jenkins.
   - `kubectl get deploy -n timer-app` — ensure both colours exist and the idle one is scaled to a single replica.
7. **Trigger an automatic redeploy**
   - Make a code change (for example, tweak UI copy in `src/App.tsx`), commit, and push to `main`.
   - GitHub webhook or polling will trigger the pipeline, automatically promoting the new colour.

## Pipeline Stages

### 1. Checkout
- Clones the repository and captures the short commit hash for notifications.

### 2. Build Docker Image
- Builds the application container inside the Minikube Docker runtime with both build-number and `latest` tags.

### 3. Prepare Manifests
- Updates `k8s/kustomization.yaml` with the freshly built image tag.
- Applies the full Kubernetes manifest set (blue, green, service, ingress, HPAs).

### 4. Blue-Green Deploy
- Detects the colour currently serving traffic.
- Updates the idle colour deployment to use the new image and waits for pods to become ready.

### 5. Smoke Test New Colour
- Port-forwards directly to a pod from the new colour and executes a HTTP health probe.

### 6. Switch Traffic
- Patches the Kubernetes service selector to route traffic to the new colour.
- Scales the previously active colour down to a single replica to keep it warm for fast rollback.

### 7. Post-Deployment Health Check
- Port-forwards the service to confirm that traffic through the new colour responds successfully.

### 8. Housekeeping
- Prunes dangling Docker images from the Minikube registry.

## Deployment Commands

### Manual Deployment
```bash
# Build and push image
docker build -t your-registry.com/timer-app:latest .
docker push your-registry.com/timer-app:latest

# Deploy to Kubernetes with both colours
kubectl apply -k k8s/
kubectl rollout status deployment/timer-app-blue -n timer-app
kubectl rollout status deployment/timer-app-green -n timer-app
```

### Rollback
```bash
kubectl patch service timer-app-service -n timer-app --type=merge -p '{"spec":{"selector":{"app":"timer-app","track":"blue"}}}'
kubectl rollout undo deployment/timer-app-green -n timer-app
```

## Monitoring

### Check Deployment Status
```bash
kubectl get all -n timer-app
kubectl describe deployment timer-app-blue -n timer-app
kubectl describe deployment timer-app-green -n timer-app
```

### View Logs
```bash
kubectl logs -n timer-app deployment/timer-app-blue
kubectl logs -n timer-app deployment/timer-app-green
kubectl logs -n timer-app -l app=timer-app
```

### Access Application
```bash
# Port forward active colour
kubectl port-forward -n timer-app service/timer-app-service 8080:80

# Directly hit the inactive colour for verification
kubectl port-forward -n timer-app deployment/timer-app-green 8081:80
curl -I http://localhost:8081/

# Via ingress (if configured)
curl -H "Host: timer-app.local" http://your-ingress-ip/
```

## Troubleshooting

### Common Issues

1. **Docker Build Fails**
   - Check Dockerfile syntax
   - Verify Node.js version compatibility
   - Check for missing dependencies

2. **Kubernetes Deployment Fails**
   - Verify kubeconfig credentials
   - Check namespace exists
   - Validate YAML syntax

3. **Health Check Fails**
   - Check pod logs
   - Verify service endpoints
   - Test port-forward manually

### Debug Commands
```bash
# Check pod status
kubectl get pods -n timer-app -o wide

# Describe pod for events
kubectl describe pod -n timer-app -l app=timer-app

# Check service endpoints
kubectl get endpoints -n timer-app

# Test connectivity
kubectl run debug --image=busybox -it --rm -- nslookup timer-app-service.timer-app.svc.cluster.local
```

## Security Considerations

1. **Use Secrets for sensitive data**
2. **Implement RBAC for Kubernetes**
3. **Scan Docker images for vulnerabilities**
4. **Use non-root containers**
5. **Implement network policies**

## Scaling

The deployment includes HPA (Horizontal Pod Autoscaler):
- Min replicas: 2
- Max replicas: 10
- CPU threshold: 70%
- Memory threshold: 80%

Monitor scaling with:
```bash
kubectl get hpa -n timer-app
kubectl describe hpa timer-app-blue-hpa -n timer-app
kubectl describe hpa timer-app-green-hpa -n timer-app
```
