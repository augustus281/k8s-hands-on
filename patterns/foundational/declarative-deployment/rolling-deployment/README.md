# ROLLING DEPLOYMENT

## What is Rolling Update?

A rolling update is a method of deploying new versions of an application without downtime. Instead of stopping entire
application to deploy the new version, Kubernetes replaces old Pods with new ones gradually.

## Practical Lab

### Step 1: Deploying the Initial Version of the Application

Defining the Deployment: Let's deploy a simple nginx version 1.28

```bash
kubectl create deployment nginx --image=nginx:1.28
kubectl annotate deployment nginx kubernetes.io/change-cause="Initial deploy nginx 1.28"
```

### Step 2: Making Changes and Deploying with a Rolling Update

**1. Modifying the Deployment**: Update the nginx image version to 1.29

```bash
kubectl set image deployment nginx nginx=nginx:1.29
kubectl annotate deployment nginx kubernetes.io/change-cause="Update nginx to 1.29"
```

**2. Check Rollout History**

```bash
kubectl rollout history deployment nginx

deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         Initial deploy nginx 1.28
2         Update nginx to 1.29
```

**3. Observing the Rolling Update**: Watch the update process in real-time

```bash
kubectl rollout status deployment nginx

deployment "nginx" successfully rolled out
```

### Step 3: Rollback

**1. Add another revision:** Set nginx version to 1.27

```bash
kubectl set image deployment nginx nginx=nginx:1.27
```

**2. Check rollout history:** We currently have 3 revisions

```bash
kubectl rollout history deployment nginx
```

**3. Rollback:** Rollback to the first revision (nginx 1.28)

```bash
kubectl rollout undo deployment nginx --to-revision=1
```

**4. Check revision history:** 4th revision is the same as 1st revision

```bash
kubectl rollout history deployment nginx   
```

