# Kubernetes Scenario-Based Pod Interview Questions

A collection of scenario-based interview questions focused on Kubernetes Pods.

## Table of Contents
1. [Multi-Container Pod Communication](#1-multi-container-pod-communication)  
2. [Pod Scheduling on GPU Nodes](#2-pod-scheduling-on-gpu-nodes)  
3. [Persistent Storage for Stateful Pods](#3-persistent-storage-for-stateful-pods)  
4. [Liveness vs Readiness Probes](#4-liveness-vs-readiness-probes)  
5. [Init Containers for Pre-Start Tasks](#5-init-containers-for-pre-start-tasks)  
6. [Resource Limits and OOMKills](#6-resource-limits-and-oomkills)  
7. [Pod Networking Issues](#7-pod-networking-issues)  
8. [Pod Security Context](#8-pod-security-context)
9. [Core Security Context Configuration](#9-Core-Security-Context-Configuration)
10. [Pod Security Standards (PSS) Compliance](#10-Pod-Security-Standards-(PSS)-Compliance)
11. [Network Hardening](#11-Network-Hardening)
12. [Runtime Security Enhancements](#12-Runtime-Security-Enhancements)
13. [Supply Chain Security](#13-Supply-Chain-Security)
14. [Implementation Checklist](#14-Implementation-Checklist)

---

### 1. Multi-Container Pod Communication <a name="1-multi-container-pod-communication"></a>
**Scenario**:  
A Pod contains a primary application container and a logging sidecar. How would you ensure they share files without persistent storage?

**Solution**:  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: app-container
    image: nginx
    volumeMounts:
    - name: shared-vol
      mountPath: /app-logs
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /logs/access.log"]
    volumeMounts:
    - name: shared-vol
      mountPath: /logs
  volumes:
  - name: shared-vol
    emptyDir: {}
```
### 2. Pod Scheduling on GPU Nodes <a name="2-pod-scheduling-on-gpu-nodes"></a>
**Scenario:**;

Ensure ML workload Pods only schedule on GPU-equipped nodes.

Implementation:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: ml-container
    image: tensorflow-gpu
    resources:
      limits:
        nvidia.com/gpu: 2
  nodeSelector:
    accelerator: gpu
```
Consider node taints/tolerations for dedicated nodes
### 3. Persistent Storage for Stateful Pods <a name="3-persistent-storage-for-stateful-pods"></a>
**Scenario:**;

Database Pod requiring persistent storage across restarts.
StatefulSet Configuration:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 1
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
```

### 4. Liveness vs Readiness Probes <a name="4-liveness-vs-readiness-probes"></a>
Scenario:

Pod crashes intermittently while receiving traffic.
Diagnostic Configuration:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20

readinessProbe:
  exec:
    command: ["/bin/sh", "-c", "test -f /tmp/ready"]
  initialDelaySeconds: 5
  periodSeconds: 5
```
Troubleshooting Flowchart:
```yaml
Pod Crash Cycle → Check Events → Verify Probe Configuration → 
Adjust Timeouts/Delays → Test Failure Thresholds
```
### 5. Init Containers for Pre-Start Tasks <a name="5-init-containers-for-pre-start-tasks"></a>
Scenario:

Configuration download required before main container starts.

Implementation:
```yaml
initContainers:
- name: config-downloader
  image: alpine/curl
  command: ['sh', '-c', 'curl -o /config/app.conf $CONFIG_URL']
  env:
  - name: CONFIG_URL
    value: "https://config-server/v1/app"
  volumeMounts:
  - name: config-volume
    mountPath: /config
```
### 6. Resource Limits and OOMKills <a name="6-resource-limits-and-oomkills"></a>
Diagnosis Workflow:
```yaml
kubectl describe pod [POD_NAME] | grep -i OOM
kubectl logs [POD_NAME] --previous
kubectl top pod [POD_NAME]
```
Optimization Strategy:

Set requests = limits for predictable behavior
Implement vertical pod autoscaling
Use memory profiling tools

### 7. Pod Networking Issues <a name="7-pod-networking-issues"></a>
Troubleshooting Matrix:
| Symptom | Check | Tool |
|----------|----------|----------|
| No inter-pod comms | Network Policies   | calicoctl   |
| DNS resolution failed   | CoreDNS Pod status   | nslookup   |
| Service unreachable   | Endpoint slices  | kubectl get ep   |

### 8. Pod Security Context <a name="8-pod-security-context"></a>
Hardened Configuration:
```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 4000
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```
### 9. Core Security Context Configuration <a name="9-Core-Security-Context-Configuration"></a>
Essential settings to enforce least privilege:
```yaml
securityContext:
  runAsUser: 1000              # Non-root user execution [[1]](#__1)
  runAsGroup: 3000             # Non-root group context
  fsGroup: 4000                # File system group ownership
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]              # Remove all Linux capabilities [[1]](#__1)
  seccompProfile:
    type: RuntimeDefault       # Default seccomp filtering [[0]](#__0)
  readOnlyRootFilesystem: true # Immutable container
```
| Key Features: |
|---------------|
| Prevents privilege escalation attacks |
| Removes unnecessary kernel capabilities |
| Enforces read-only root filesystem where possible |

### 10. Pod Security Standards (PSS) Compliance  <a name="10-Pod-Security-Standards-(PSS)-Compliance"></a>

**Must validate against latest v1.25+ PSS standards**

**Requires all containers to:**
- Run as non-root users
- Prohibit privileged containers
- Drop ALL capabilities
- Enforce read-only root filesystems

### 11. Network Hardening <a name= "11-Network-Hardening"></a>

Network Policy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Best Practices:**
- Default deny all ingress/egress traffic 
- Whitelist required communication paths
- Use service mesh for mTLS encryption

### 12.Runtime Security Enhancements <a name= "12-Runtime-Security-Enhancements"> </a>
Advanced protection mechanisms:
```yaml
securityContext:
  apparmorProfile:
    type: runtime/default
  sysctls:
  - name: kernel.perf_event_paranoid
    value: "3"
```

**Protection Features:**
- AppArmor/SELinux profile enforcement
- Kernel parameter hardening
- eBPF-based runtime security monitoring

### 13. Supply Chain Security  <a name= "13-Supply-Chain-Security"></a>
Critical verification steps:
```yaml
# Example Admission Controller Config
apiVersion: policy/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: verify-images
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
  validations:
    - expression: "object.spec.containers.all(c, c.image.endsWith('@sha256'))"
      message: "All images must use digest references"
```

**Verification Requirements:**
- Signed container images with Cosign 
- SBOM (Software Bill of Materials) validation
- Vulnerability scanning integration

### 14. Implementation Checklist <a name= " 14-Implementation-Checklist"></a>

**Checklist**
- Apply Restricted Pod Security Admission 
- Configure granular securityContext settings 
- Enforce network segmentation policies 
- Enable encryption for secrets/data at rest 
- Implement image signing/verification workflows

Audit Command:
```yaml
kubectl audit-pod <pod-name> --security-checks=all
```
