# 🎯 Jenkins Pipeline Setup Guide
# Avoid Red X's - Get Green Checkmarks!

## 📋 **Step-by-Step Jenkins Configuration**

### **Step 1: Access Jenkins Dashboard**
```
URL: http://localhost:8080
Login: Use your Jenkins credentials
```

### **Step 2: Create New Pipeline Job**
1. **Click:** "New Item" (top left)
2. **Name:** `timer-app-pipeline`
3. **Type:** Pipeline
4. **Click:** OK

### **Step 3: Configure Pipeline Settings**
```
General Tab:
✅ Description: CI/CD Pipeline for Timer App with Blue-Green Deployment

Pipeline Tab:
✅ Definition: Pipeline script from SCM
✅ SCM: Git
✅ Repository URL: https://github.com/ajayabd17/TaskTImer-React.git
✅ Branch: */main
✅ Script Path: Jenkinsfile
✅ Credentials: github-creds (if needed)

Build Triggers Tab:
✅ GitHub hook trigger for GITScm polling
✅ Poll SCM: H/5 * * * * (every 5 minutes)

Save Configuration
```

### **Step 4: Verify Credentials**
Make sure these credentials exist in Jenkins:
```
✅ kubeconfig (jenkins-kubeconfig.yaml)
✅ github-creds (if using private repo)
```

## 🚀 **Expected Successful Dashboard View**

### **Pipeline Status**
```
┌─────────────────────────────────────────────────────────┐
│ 🚀 timer-app-pipeline                    ✅ SUCCESS      │
├─────────────────────────────────────────────────────────┤
│ Permalinks:                                              │
│ • Last build (#15), 2 min ago                           │
│ • Last successful build (#15), 2 min ago               │
│ • Last stable build (#15), 2 min ago                   │
│ • Last completed build (#15), 2 min ago                │
└─────────────────────────────────────────────────────────┘
```

### **Build History**
```
Builds Widget:
Today:
✅ #15 22:45  (Green checkmark)
✅ #14 22:30  (Green checkmark)
✅ #13 22:15  (Green checkmark)
✅ #12 22:00  (Green checkmark)
```

### **Pipeline Stages (Successful)**
```
✅ Checkout (1s)
✅ Build Docker Image (3s)
✅ Prepare Manifests (30s)
✅ Blue-Green Deploy (2s)
✅ Smoke Test New Color (45s)
✅ Switch Traffic (20s)
✅ Post-Deployment Health Check (30s)
✅ Housekeeping (12s)
```

## 🔧 **Troubleshooting Failed Builds**

### **Common Issues & Solutions**

**Issue 1: kubectl not found**
```
Error: kubectl: command not found
Solution: Install kubectl on Jenkins agent
```

**Issue 2: Docker daemon connection**
```
Error: Cannot connect to Docker daemon
Solution: Configure Docker socket access
```

**Issue 3: Credentials not found**
```
Error: Could not find credentials
Solution: Verify credential IDs match Jenkinsfile
```

**Issue 4: Kubernetes namespace not found**
```
Error: namespace "timer-app" not found
Solution: Ensure namespace exists
```

## 🎯 **Quick Test Commands**

### **Test Jenkins Pipeline Locally**
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

## 🎓 **For Professor Demonstration**

### **Show This Sequence:**
1. **Jenkins Dashboard** - Professional pipeline view
2. **Build History** - All green checkmarks
3. **Live Build** - Click "Build Now"
4. **Stage Execution** - Real-time progress
5. **Successful Completion** - Green status
6. **Application Verification** - Timer app running

### **Key Points to Highlight:**
- ✅ **No red X's** - All builds successful
- ✅ **Complete automation** - No manual intervention
- ✅ **Blue-green deployment** - Zero downtime
- ✅ **Health verification** - Automatic testing
- ✅ **Production ready** - Enterprise features

## 🚀 **Success Criteria**

Your pipeline is working when you see:
- ✅ **Green checkmarks** in build history
- ✅ **"SUCCESS" status** in pipeline view
- ✅ **All stages completed** successfully
- ✅ **Application deployed** and healthy
- ✅ **Blue-green switching** working

## 📋 **Final Checklist**

- [ ] Jenkins running on port 8080
- [ ] Timer app running on port 3000
- [ ] Credentials uploaded to Jenkins
- [ ] Pipeline job created
- [ ] Jenkinsfile configured
- [ ] First build successful
- [ ] Application health verified

**🎉 Ready for Professor Demonstration with Green Checkmarks!**
