# Jenkins CI/CD Setup Guide for Timer App

## 🎯 Prerequisites Completed ✅
- ✅ Kubernetes cluster (Minikube) running
- ✅ Blue-Green deployments configured
- ✅ Jenkins-compatible kubeconfig created
- ✅ Docker images built and tested

## 📁 Files Ready for Jenkins
- `jenkins-kubeconfig.yaml` - Kubernetes credentials (812 bytes)
- `Jenkinsfile` - Complete CI/CD pipeline
- `k8s/` directory - All Kubernetes manifests

## 🔐 Jenkins Credentials Setup

### Step 1: Access Jenkins
1. Open browser: `http://localhost:8080`
2. Login with your Jenkins credentials

### Step 2: Add Kubernetes Credentials
1. Go to: **Manage Jenkins** → **Manage Credentials**
2. Click: **"Add Credentials"**
3. Configure:
   ```
   Kind: Secret file
   Scope: Global
   File: Upload jenkins-kubeconfig.yaml
   ID: kubeconfig
   Description: Kubernetes config for minikube cluster
   ```

### Step 3: Add Docker Credentials (Optional)
1. Click: **"Add Credentials"** again
2. Configure:
   ```
   Kind: Username with password
   Scope: Global
   Username: ajayabd17
   Password: [your-dockerhub-password]
   ID: docker-registry-credentials
   Description: Docker Hub credentials
   ```

## 🚀 Pipeline Configuration

### Step 1: Create Pipeline Job
1. **New Item** → **Pipeline** → Name: `timer-app-deployment`
2. **Pipeline** → **Definition**: `Pipeline script from SCM`
3. **SCM**: Git
4. **Repository URL**: `https://github.com/ajayabd17/TaskTImer-React.git`
5. **Branch**: `*/main`
6. **Script Path**: `Jenkinsfile`

### Step 2: Configure Build Triggers
- ✅ **GitHub hook trigger for GITScm polling**
- ✅ **Poll SCM**: `H/5 * * * *` (every 5 minutes)

## 🧪 Testing Your Pipeline

### Manual Test Commands
```bash
# Test kubeconfig
kubectl --kubeconfig=jenkins-kubeconfig.yaml get nodes

# Test application
kubectl --kubeconfig=jenkins-kubeconfig.yaml get all -n timer-app

# Test Docker build
eval $(minikube docker-env)
docker build -t timer-app:test .

# Test blue-green switch
kubectl --kubeconfig=jenkins-kubeconfig.yaml patch service timer-app-service -n timer-app --type=merge -p '{"spec":{"selector":{"app":"timer-app","track":"green"}}}'
```

## 📊 Expected Jenkins Output

### Successful Build Console Output:
```
Started by user ajayabd17
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/timer-app-deployment
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Checkout)
[Pipeline] checkout
Cloning the remote Git repository
Cloning repository https://github.com/ajayabd17/TaskTImer-React.git
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ git rev-parse --short HEAD
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Build Docker Image)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ eval $(minikube docker-env)
+ docker build -t timer-app:42 .
+ docker build -t timer-app:latest .
Sending build context to Docker daemon  153.6kB
Step 1/7 : FROM node:20-alpine AS builder
 ---> 6178e78b972f7
Step 2/7 : WORKDIR /app
 ---> Running in 1234567890ab
Step 3/7 : COPY package*.json ./
 ---> 1234567890ab
Step 4/7 : RUN npm ci
 ---> Running in 1234567890ab
added 197 packages, and audited 198 packages in 17s
Step 5/7 : COPY . .
 ---> 1234567890ab
Step 6/7 : RUN rm -rf node_modules package-lock.json && npm install
 ---> Running in 1234567890ab
added 195 packages, and audited 196 packages in 20s
Step 7/7 : RUN npm run build
 ---> Running in 1234567890ab
> my-app@0.0.0 build
> tsc -b && vite build
rolldown-vite v7.1.14 building for production...
✓ built in 168ms
Successfully built 8918e94f9050
Successfully tagged timer-app:42
Successfully tagged timer-app:latest
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Blue-Green Deploy)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ kubectl get service timer-app-service -n timer-app -o jsonpath='{.spec.selector.track}' 2>/dev/null || true
[Pipeline] sh
Active color: blue -> Target color: green
+ kubectl set image deployment/timer-app-green timer-app=timer-app:42 -n timer-app
deployment.apps/timer-app-green image updated
+ kubectl rollout status deployment/timer-app-green -n timer-app --timeout=300s
Waiting for deployment "timer-app-green" rollout to finish: 0 out of 3 new replicas have been updated...
Waiting for deployment "timer-app-green" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "timer-app-green" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "timer-app-green" successfully rolled out
+ kubectl wait --for=condition=available deployment/timer-app-green -n timer-app --timeout=300s
deployment.apps/timer-app-green condition met
+ kubectl wait --for=condition=ready pod -l app=timer-app,track=green -n timer-app --timeout=300s
pod/timer-app-green-abc123 condition met
pod/timer-app-green-def456 condition met
pod/timer-app-green-ghi789 condition met
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Smoke Test New Color)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ set -e
+ kubectl get pods -n timer-app -l app=timer-app,track=green -o jsonpath='{.items[0].metadata.name}'
+ kubectl port-forward -n timer-app pod/timer-app-green-abc123 3001:80 >/tmp/timer-app-bluegreen-42.log 2>&1 &
+ sleep 10
+ curl -f http://localhost:3001
+ kill 12345
+ wait 12345 || true
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Switch Traffic)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ kubectl patch service timer-app-service -n timer-app --type=merge -p '{"spec":{"selector":{"app":"timer-app","track":"green"}}}'
service/timer-app-service patched
+ kubectl scale deployment/timer-app-blue -n timer-app --replicas=1
deployment.apps/timer-app-blue scaled
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Post-Deployment Health Check)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ set -e
+ timeout 30 kubectl port-forward -n timer-app service/timer-app-service 3000:80 >/tmp/timer-app-service-42.log 2>&1 &
+ sleep 10
+ curl -f http://localhost:3000
+ kill 67890
+ wait 67890 || true
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Housekeeping)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ eval $(minikube docker-env)
+ docker image prune -f
Deleted Images:
deleted: sha256:oldimage123
deleted: sha256:oldimage456
Total reclaimed space: 45.2MB
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] }
[Pipeline] End of Pipeline
✅ Pipeline completed successfully!
🚀 Timer app deployed to green with image: timer-app:42
```

## 🎯 For Professor Demonstration

### What to Show:
1. **Jenkins Dashboard** - Show successful builds
2. **Pipeline Configuration** - Show Jenkinsfile stages
3. **Live Build** - Trigger "Build Now" and show real-time progress
4. **Blue-Green Deployment** - Show traffic switching
5. **Application Health** - Show HTTP 200 responses
6. **Kubernetes Resources** - Show pods, services, HPA

### Key Points to Highlight:
- ✅ **Zero-downtime deployments**
- ✅ **Automatic health verification**
- ✅ **Intelligent traffic switching**
- ✅ **Complete CI/CD automation**
- ✅ **Production-ready monitoring**

## 🔧 Troubleshooting

### Common Issues:
1. **kubectl not found**: Install kubectl on Jenkins agent
2. **Docker daemon**: Configure Docker socket access
3. **Credentials not found**: Verify credential IDs match Jenkinsfile
4. **Namespace not found**: Ensure timer-app namespace exists

### Debug Commands:
```bash
# Check cluster status
kubectl --kubeconfig=jenkins-kubeconfig.yaml cluster-info

# Check application status
kubectl --kubeconfig=jenkins-kubeconfig.yaml get all -n timer-app

# Check service routing
kubectl --kubeconfig=jenkins-kubeconfig.yaml get service timer-app-service -n timer-app -o jsonpath='{.spec.selector.track}'
```

## 🚀 Success Criteria

Your pipeline is working when you see:
- ✅ **Green build status** in Jenkins
- ✅ **All stages completed** successfully
- ✅ **Traffic switched** between blue and green
- ✅ **Application responding** with HTTP 200
- ✅ **HPA configured** for both environments

**Ready for Professor Demonstration!** 🎉
