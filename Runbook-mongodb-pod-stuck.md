Good call—let’s format it properly for GitHub README (clean Markdown + readable).

Copy-paste this directly into your repo:

---

```markdown
# Runbook: Resolving MongoDB Pod Stuck in Pending / ImagePullBackOff (Kubernetes - Kind)

## Environment
- Kubernetes: Kind (local cluster)
- Namespace: `easyshop`
- Application: EasyShop (Next.js) + MongoDB (StatefulSet)

---

## Initial Problem

### Symptoms Observed
- MongoDB pod stuck in:
  - `Pending`
  - `ContainerCreating`
  - `ImagePullBackOff`

- PVC status:
```

STATUS: Pending
STORAGECLASS: gp2

```

- Events:
```

pod has unbound immediate PersistentVolumeClaims
AttachVolume.Attach failed (AWS EBS)

````

---

## Root Cause Analysis

### 1. Storage Issue
- PV used: `awsElasticBlockStore` (AWS-only)
- Cluster: Kind (local)

**Result:**
- Volume could not attach
- PVC remained `Pending`

---

### 2. StorageClass Mismatch
- PV:
```yaml
storageClassName: manual
````

* PVC:

  ```yaml
  storageClassName: gp2
  ```

**Result:**

* No binding due to mismatch

---

### 3. StatefulSet Limitation

Attempted to update:

* `volumeClaimTemplates`

**Error:**

```
updates to statefulset spec are forbidden
```

---

### 4. Image Pull Issue

```
ImagePullBackOff
500 Internal Server Error from docker.io
```

**Result:**

* Temporary registry/network issue

---

## Resolution Steps

### Step 1: Replace AWS PV with Local PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/data/mongodb
  persistentVolumeReclaimPolicy: Retain
```

---

### Step 2: Fix StatefulSet StorageClass

```yaml
volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: manual
      resources:
        requests:
          storage: 5Gi
```

---

### Step 3: Delete & Recreate StatefulSet

```bash
kubectl delete statefulset mongodb -n easyshop
kubectl delete pvc --all -n easyshop
kubectl apply -f 07-mongodb-statefulset.yaml
```

---

### Step 4: Verify PVC Binding

```bash
kubectl get pvc -n easyshop
```

Expected:

```
STATUS: Bound
STORAGECLASS: manual
```

---

### Step 5: Handle ImagePullBackOff

* Observed: Docker Hub error (500)
* Action: Wait for retry

Result:

```
Successfully pulled image "mongo:6.0"
```

---

### Step 6: Validate Pod

```bash
kubectl get pods -n easyshop
```

```
mongodb-0   Running
```

---

### Step 7: Verify MongoDB Access

```bash
kubectl exec -it mongodb-0 -n easyshop -- mongosh
```

---

## Warnings

### Access Control Disabled

```
Access control is not enabled
```

* No authentication (OK for local)

---

### Kernel Warning

```
vm.max_map_count is too low
```

* Ignore in local
* Fix in production

---

## Key Learnings

### Storage

* PV & PVC must match:

  * StorageClass
  * AccessModes
  * Capacity

### StatefulSet

* Cannot modify `volumeClaimTemplates`
* Must recreate resource

### Image Pull

* `ImagePullBackOff` can be temporary
* Kubernetes retries automatically

---

## Final Outcome

| Component   | Status   |
| ----------- | -------- |
| MongoDB Pod | Running  |
| PVC         | Bound    |
| PV          | Attached |
| App         | Running  |
| DB Access   | Verified |

---

## Useful Commands

```bash
kubectl get pods -n easyshop
kubectl get pvc -n easyshop
kubectl describe pod mongodb-0 -n easyshop
kubectl logs -l app=easyshop -n easyshop
kubectl exec -it mongodb-0 -n easyshop -- mongosh
```

---

## Conclusion

Issue resolved by:

* Replacing AWS storage with local storage
* Fixing StorageClass mismatch
* Recreating StatefulSet
* Allowing Kubernetes to recover image pull

System is stable and working.

```
