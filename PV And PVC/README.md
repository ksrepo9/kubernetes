# Kubernetes PV & PVC Scenario-Based Interview Questions and Answers


## Table of Contents
- [Scenario 1: PVC Pending State](#scenario-1-pvc-pending-state)
- [Scenario 2: PV Access Modes](#scenario-2-pv-access-modes)
- [Scenario 3: Storage Class Dynamics](#scenario-3-storage-class-dynamics)
- [Scenario 4: Volume Expansion](#scenario-4-volume-expansion)
- [Scenario 5: Data Protection & Backup](#scenario-5-data-protection--backup)
- [Scenario 6: Multi-Pod Access](#scenario-6-multi-pod-access)
- [Scenario 7: Performance Issues](#scenario-7-performance-issues)
- [Scenario 8: StatefulSet Volumes](#scenario-8-statefulset-volumes)
- [Scenario 9: Volume Snapshots](#scenario-9-volume-snapshots)
- [Scenario 10: Cross-Namespace Volume Sharing](#scenario-10-cross-namespace-volume-sharing)

---

## Scenario 1: PVC Pending State

**Question**: A PersistentVolumeClaim is stuck in "Pending" status. What are the possible causes and how would you troubleshoot this?

**Answer**:

### Possible Causes:
1. No available PersistentVolume matching the PVC requirements
2. StorageClass not found or misconfigured
3. Insufficient storage capacity
4. Access mode mismatch
5. Selector/label mismatches

### Troubleshooting Steps:
```bash
# Check PVC status and events
kubectl describe pvc <pvc-name>

# List available PVs
kubectl get pv

# Check StorageClasses
kubectl get storageclass

# Check PVC details
kubectl get pvc <pvc-name> -o yaml

# Check provisioner logs if using dynamic provisioning
kubectl logs -n kube-system -l app=ebs-csi-controller
```

### Common Solutions:
```yaml
# Example of proper PVC configuration
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd  # Ensure this exists
```

---

## Scenario 2: PV Access Modes

**Question**: You need to deploy a database that requires multiple pods to read and write to the same volume. Which access mode should you use and why?

**Answer**:

### Access Mode Selection:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-db-pvc
spec:
  accessModes:
    - ReadWriteMany  # Allows multiple pods to read and write
  resources:
    requests:
      storage: 100Gi
  storageClassName: nfs-storage
```

### Access Mode Comparison:
| Access Mode | Description | Use Cases |
|-------------|-------------|-----------|
| ReadWriteOnce | Single node read-write | Single pod databases |
| ReadOnlyMany | Multiple nodes read-only | Config maps, static content |
| ReadWriteMany | Multiple nodes read-write | Shared databases, file sharing |
| ReadWriteOncePod | Single pod read-write (K8s 1.22+) | Critical single-pod data |

### Storage Backends Supporting ReadWriteMany:
- NFS
- CephFS
- Azure File
- GlusterFS
- Some cloud provider file storage

---

## Scenario 3: Storage Class Dynamics

**Question**: How does dynamic provisioning work, and what happens when you delete a PVC with `delete` reclaim policy?

**Answer**:

### Dynamic Provisioning Flow:
1. User creates PVC with specific StorageClass
2. PVC controller detects no matching PV exists
3. StorageClass provisioner creates the actual storage
4. Provisioner creates PV and binds to PVC
5. Pod can now use the volume

### Reclaim Policies:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Delete  # Options: Delete, Retain, Recycle
  # ...
```

### Delete Reclaim Policy Behavior:
```bash
# When PVC is deleted with reclaimPolicy: Delete
kubectl delete pvc my-pvc

# What happens:
# 1. PV is deleted
# 2. Underlying storage resource is deleted
# 3. All data is lost
```

### Retain Policy Example:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: important-data-pv
spec:
  persistentVolumeReclaimPolicy: Retain  # Manual cleanup required
  capacity:
    storage: 50Gi
  # ...
```

---

## Scenario 4: Volume Expansion

**Question**: Your application is running out of disk space. How can you expand a PVC without downtime?

**Answer**:

### Prerequisites:
1. StorageClass must support expansion: `allowVolumeExpansion: true`
2. Underlying storage provider must support resizing

### Step-by-Step Expansion:
```yaml
# 1. Check if StorageClass allows expansion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-ssd
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true  # This must be true
```

```bash
# 2. Modify PVC to request more storage
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# 3. Monitor expansion progress
kubectl get pvc my-pvc -w

# 4. Verify file system expansion inside pod
kubectl exec -it <pod-name> -- df -h
```

### File System Expansion Inside Pod:
```bash
# For XFS filesystems
xfs_growfs /data

# For ext4 filesystems
resize2fs /dev/sdb
```

---

## Scenario 5: Data Protection & Backup

**Question**: How would you implement a backup strategy for persistent volumes in production?

**Answer**:

### Strategy 1: Volume Snapshots
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: database-pvc
```

### Strategy 2: Application-Level Backups
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:13
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME > /backup/db-$(date +%Y%m%d).sql
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
            - name: database-volume
              mountPath: /var/lib/postgresql/data
              readOnly: true
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          - name: database-volume
            persistentVolumeClaim:
              claimName: database-pvc
          restartPolicy: OnFailure
```

### Strategy 3: Velero for Cluster Backups
```bash
# Backup specific namespace with PVs
velero backup create app-backup --include-namespaces production

# Schedule regular backups
velero create schedule daily-backup --schedule="0 1 * * *"
```

---

## Scenario 6: Multi-Pod Access

**Question**: You need multiple pods to access the same configuration data. How would you design this?

**Answer**:

### Solution 1: ReadOnlyMany PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: config-pvc
spec:
  accessModes:
  - ReadOnlyMany  # Multiple pods can read
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-storage
```

### Solution 2: Init Container for Data Population
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  initContainers:
  - name: load-config
    image: busybox
    command: ['sh', '-c', 'cp /config-source/* /shared-config/']
    volumeMounts:
    - name: config-source
      mountPath: /config-source
    - name: shared-config
      mountPath: /shared-config
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared-config
      mountPath: /etc/app-config
      readOnly: true
  volumes:
  - name: config-source
    configMap:
      name: app-config
  - name: shared-config
    emptyDir: {}
```

### Solution 3: Sidecar Pattern for Dynamic Updates
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-aware-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        volumeMounts:
        - name: shared-config
          mountPath: /config
      - name: config-sync
        image: config-sync:latest
        volumeMounts:
        - name: shared-config
          mountPath: /sync-target
        - name: config-source
          mountPath: /source
      volumes:
      - name: shared-config
        emptyDir: {}
      - name: config-source
        persistentVolumeClaim:
          claimName: config-pvc
```

---

## Scenario 7: Performance Issues

**Question**: Users are complaining about slow I/O on database pods. How would you diagnose and resolve storage performance issues?

**Answer**:

### Diagnosis Steps:
```bash
# Check PVC/PV configuration
kubectl get pvc -o wide
kubectl describe pv <pv-name>

# Check storage class parameters
kubectl describe storageclass <storage-class>

# Monitor I/O metrics
kubectl top pod --containers

# Check node disk performance
kubectl debug node/<node-name> --image=busybox -- df -h
```

### Performance Optimization:

#### 1. Storage Class Tuning:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: ebs.csi.aws.com
parameters:
  type: io2  # Provisioned IOPS SSD
  iopsPerGB: "50"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

#### 2. Pod Resource Limits for I/O:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
spec:
  containers:
  - name: database
    image: mysql:8.0
    resources:
      requests:
        memory: "4Gi"
        cpu: "2"
      limits:
        memory: "8Gi"
        cpu: "4"
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

#### 3. Filesystem Optimization:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-pod
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: fast-pvc
    # Add mount options for performance
    # mountOptions:
    #   - noatime
    #   - nodiratime
```

---

## Scenario 8: StatefulSet Volumes

**Question**: How do PersistentVolumeClaims work with StatefulSets, and what are the advantages?

**Answer**:

### StatefulSet PVC Template:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:  # PVC template for each pod
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "ssd"
      resources:
        requests:
          storage: 10Gi
```

### Advantages:
1. **Predictable Naming**: PVCs get names like `www-web-0`, `www-web-1`, etc.
2. **Stable Identity**: Each pod maintains its own persistent storage
3. **Ordered Operations**: Pods are created/deleted in order
4. **Independent Lifecycle**: PVCs persist even if pods are deleted

### PVC Management:
```bash
# List PVCs created by StatefulSet
kubectl get pvc -l app=nginx

# Scale down (PVCs are retained)
kubectl scale statefulset web --replicas=2

# Scale up (new PVCs created from template)
kubectl scale statefulset web --replicas=4
```

---

## Scenario 9: Volume Snapshots

**Question**: How would you create and restore from volume snapshots in Kubernetes?

**Answer**:

### 1. Create VolumeSnapshotClass:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  # Optional provider-specific parameters
```

### 2. Create Snapshot:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot
spec:
  volumeSnapshotClassName: csi-aws-snapshot-class
  source:
    persistentVolumeClaimName: database-pvc
```

### 3. Restore from Snapshot:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd
  dataSource:
    name: db-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  resources:
    requests:
      storage: 20Gi  # Must be >= snapshot size
```

### Snapshot Operations:
```bash
# Check snapshot status
kubectl get volumesnapshot
kubectl describe volumesnapshot db-snapshot

# List snapshot contents
kubectl get volumesnapshotcontent

# Delete snapshot
kubectl delete volumesnapshot db-snapshot
```

---

## Scenario 10: Cross-Namespace Volume Sharing

**Question**: How can you share a PersistentVolume across multiple namespaces?

**Answer**:

### Method 1: Cluster-wide ReadOnlyMany PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-config-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadOnlyMany  # Multiple namespaces can read
  persistentVolumeReclaimPolicy: Retain
  storageClassName: shared-storage
  nfs:
    server: nfs-server.example.com
    path: "/shared/config"
```

### Method 2: PVC in Each Namespace (Same Storage Backend)
```yaml
# In namespace A
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: team-a
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: shared-nfs
  volumeName: shared-data-pv  # Reference specific PV
```

### Method 3: CSI Volume Sharing with RBAC
```yaml
# Create ClusterRole for volume access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: volume-share-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# Bind to namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: volume-share-binding
  namespace: team-b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: volume-share-role
subjects:
- kind: ServiceAccount
  name: default
  namespace: team-b
```

### Method 4: Static Provisioning with Global Mount
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: global-share-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /exports/global
    server: nfs-server.internal
  # Important: No storageClassName for static PVs
  # This allows any namespace to claim it with matching parameters
```

---

## Best Practices Summary

### 1. Always Use StorageClasses
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gold
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 2. Set Proper Resource Requests
```yaml
resources:
  requests:
    storage: 10Gi  # Don't undersize
  # No limits for storage (handled by underlying system)
```

### 3. Use Volume Snapshots for Backups
```bash
# Regular snapshot schedule
kubectl create volumesnapshot --schedule="0 2 * * *" --source-pvc=production-db
```

### 4. Monitor Volume Usage
```bash
# Install metrics server for storage metrics
kubectl top pods --containers
```
