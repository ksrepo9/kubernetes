# Kubernetes RBAC Scenario-Based Interview Questions and Answers


## Table of Contents
- [Scenario 1: Basic RBAC Setup](#scenario-1-basic-rbac-setup)
- [Scenario 2: Namespace Isolation](#scenario-2-namespace-isolation)
- [Scenario 3: Pod Security Escalation](#scenario-3-pod-security-escalation)
- [Scenario 4: Cross-Namespace Access](#scenario-4-cross-namespace-access)
- [Scenario 5: Service Account Permissions](#scenario-5-service-account-permissions)
- [Scenario 6: Cluster-wide Operations](#scenario-6-cluster-wide-operations)
- [Scenario 7: RBAC Troubleshooting](#scenario-7-rbac-troubleshooting)
- [Scenario 8: Custom Resource Definitions](#scenario-8-custom-resource-definitions)
- [Scenario 9: Multi-team Cluster](#scenario-9-multi-team-cluster)
- [Scenario 10: Audit and Compliance](#scenario-10-audit-and-compliance)

---

## Scenario 1: Basic RBAC Setup

**Question**: A new developer needs read-only access to all resources in the "development" namespace. How would you implement this?

**Answer**:

### Step 1: Create Role with Read-Only Permissions
```yaml
# development-readonly-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-readonly
rules:
- apiGroups: [""]  # Core API group
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]  # Apps API group
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]  # Batch API group
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]  # Networking API group
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
```

### Step 2: Create RoleBinding for the User
```yaml
# development-readonly-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: development
subjects:
- kind: User
  name: "alice@company.com"  # User from your identity provider
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Step 3: Apply and Verify
```bash
# Apply the RBAC configuration
kubectl apply -f development-readonly-role.yaml
kubectl apply -f development-readonly-binding.yaml

# Verify the role was created
kubectl get role -n development

# Verify the binding was created
kubectl get rolebinding -n development

# Test access (as the user)
kubectl auth can-i get pods --namespace development
kubectl auth can-i create deployments --namespace development  # Should return 'no'
```

---

## Scenario 2: Namespace Isolation

**Question**: You have two teams, "team-a" and "team-b", that should not be able to see or access each other's resources. How would you implement namespace isolation?

**Answer**:

### Team A Namespace and RBAC
```yaml
# team-a-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    team: a
---
# team-a-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-a-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# team-a-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-a
  name: team-a-admin-binding
subjects:
- kind: Group
  name: "team-a"  # Group from identity provider
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-a-admin
  apiGroup: rbac.authorization.k8s.io
```

### Team B Namespace and RBAC
```yaml
# team-b-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-b
  labels:
    team: b
---
# team-b-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-b
  name: team-b-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# team-b-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: team-b
  name: team-b-admin-binding
subjects:
- kind: Group
  name: "team-b"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-b-admin
  apiGroup: rbac.authorization.k8s.io
```

### Network Policies for Additional Isolation
```yaml
# team-a-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-team-traffic
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          team: a
```

---

## Scenario 3: Pod Security Escalation

**Question**: A pod needs to list nodes in the cluster but should not have any other cluster-level permissions. How would you implement this securely?

**Answer**:

### Secure ClusterRole Definition
```yaml
# node-reader-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]  # Minimal required permissions
  # Note: No 'create', 'update', 'patch', 'delete'
```

### Service Account for the Pod
```yaml
# monitoring-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-monitor
  namespace: monitoring
---
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-monitor-binding
subjects:
- kind: ServiceAccount
  name: node-monitor
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Deployment Using the Service Account
```yaml
# monitoring-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-monitor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-monitor
  template:
    metadata:
      labels:
        app: cluster-monitor
    spec:
      serviceAccountName: node-monitor  # Uses the restricted service account
      containers:
      - name: monitor
        image: company/monitoring-agent:latest
        env:
        - name: CLUSTER_NAME
          value: "production"
```

### Verification Commands
```bash
# Test the service account permissions
kubectl auth can-i get nodes --as=system:serviceaccount:monitoring:node-monitor
kubectl auth can-i create pods --as=system:serviceaccount:monitoring:node-monitor  # Should be 'no'

# Check what the service account can do
kubectl create token node-monitor -n monitoring > token.txt
kubectl --token=$(cat token.txt) get nodes
kubectl --token=$(cat token.txt) get pods  # Should fail
```

---

## Scenario 4: Cross-Namespace Access

**Question**: A service in the "frontend" namespace needs to read configmaps from the "backend" namespace. How would you enable this securely?

**Answer**:

### Option 1: Cross-Namespace Role and Binding
```yaml
# backend-config-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: backend
  name: config-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  resourceNames: ["app-config"]  # Specific configmap only
---
# frontend-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend-service
  namespace: frontend
---
# cross-namespace-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: backend  # Binding in the TARGET namespace
  name: frontend-config-reader
subjects:
- kind: ServiceAccount
  name: frontend-service
  namespace: frontend  # Source namespace
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

### Option 2: ClusterRole for Cross-Namespace Access
```yaml
# cross-namespace-config-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cross-namespace-config-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
# cluster-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: frontend-to-backend-config
subjects:
- kind: ServiceAccount
  name: frontend-service
  namespace: frontend
roleRef:
  kind: ClusterRole
  name: cross-namespace-config-reader
  apiGroup: rbac.authorization.k8s.io
```

### Frontend Deployment Using the Service Account
```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      serviceAccountName: frontend-service
      containers:
      - name: frontend
        image: nginx:latest
        env:
        - name: BACKEND_CONFIG
          valueFrom:
            configMapKeyRef:
              name: app-config
              namespace: backend  # References configmap from another namespace
              key: api-url
```

---

## Scenario 5: Service Account Permissions

**Question**: A CI/CD pipeline service account needs to deploy applications to specific namespaces but shouldn't be able to modify cluster-wide resources. How would you configure this?

**Answer**:

### CI/CD Service Account with Limited Permissions
```yaml
# cicd-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: cicd
---
# cicd-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer
rules:
# Allow listing namespaces to discover where to deploy
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]

# Allow creating/updating resources in target namespaces
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
# cicd-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cicd-deployer-binding
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: cicd
roleRef:
  kind: ClusterRole
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Namespace-specific Restrictions
```yaml
# deny-sensitive-namespaces.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deny-sensitive-namespaces
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# Use Kubernetes RBAC aggregation for fine-grained control
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer-refined
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/cicd: "true"
rules: []  # Rules are automatically filled
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-app-deployment
  labels:
    rbac.example.com/cicd: "true"
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["*"]
```

---

## Scenario 6: Cluster-wide Operations

**Question**: A cluster administrator needs to grant someone permission to view all resources across the cluster but not make any changes. How would you implement this?

**Answer**:

### Cluster-wide Read-Only Access
```yaml
# cluster-readonly-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-readonly
rules:
# Core resources
- apiGroups: [""]
  resources: 
    - "pods"
    - "services"
    - "configmaps"
    - "secrets"
    - "namespaces"
    - "nodes"
    - "persistentvolumes"
    - "persistentvolumeclaims"
  verbs: ["get", "list", "watch"]

# Apps resources
- apiGroups: ["apps"]
  resources: 
    - "deployments"
    - "replicasets"
    - "statefulsets"
    - "daemonsets"
  verbs: ["get", "list", "watch"]

# Batch resources
- apiGroups: ["batch"]
  resources: 
    - "jobs"
    - "cronjobs"
  verbs: ["get", "list", "watch"]

# Networking resources
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]

# Storage resources
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]

# RBAC resources (read their own permissions)
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: 
    - "clusterroles"
    - "clusterrolebindings"
    - "roles"
    - "rolebindings"
  verbs: ["get", "list", "watch"]
---
# cluster-readonly-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-readonly-binding
subjects:
- kind: User
  name: "auditor@company.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Using Built-in ClusterRoles
```bash
# Kubernetes provides built-in cluster roles
kubectl get clusterrole | grep view

# Use system:aggregate-to-view which aggregates read-only permissions
kubectl describe clusterrole system:aggregate-to-view

# Bind user to the view clusterrole in specific namespaces
kubectl create clusterrolebinding cluster-auditor \
  --clusterrole=view \
  --user=auditor@company.com
```

### Verification
```bash
# Test permissions
kubectl auth can-i get pods --all-namespaces --as=auditor@company.com
kubectl auth can-i create deployments --as=auditor@company.com  # Should be 'no'
kubectl auth can-i delete nodes --as=auditor@company.com  # Should be 'no'

# List all resources the user can access
kubectl auth can-i --list --all-namespaces --as=auditor@company.com
```

---

## Scenario 7: RBAC Troubleshooting

**Question**: A user reports they cannot create pods in their namespace. How would you troubleshoot this RBAC issue?

**Answer**:

### Step-by-Step Troubleshooting

#### 1. Check Current Permissions
```bash
# Check if user can create pods
kubectl auth can-i create pods --as=user@company.com --namespace=development

# Check all permissions for the user
kubectl auth can-i --list --as=user@company.com --namespace=development

# Check effective permissions with verbose output
kubectl --v=8 auth can-i create pods --as=user@company.com 2>&1 | grep -i rbac
```

#### 2. Investigate RoleBindings and ClusterRoleBindings
```bash
# List all rolebindings in the namespace
kubectl get rolebindings -n development

# List all clusterrolebindings
kubectl get clusterrolebindings

# Check what subjects are bound to roles
kubectl describe rolebinding -n development
kubectl describe clusterrolebinding

# Search for bindings that include the user
kubectl get rolebindings --all-namespaces -o yaml | grep -A 5 -B 5 "user@company.com"
kubectl get clusterrolebindings -o yaml | grep -A 5 -B 5 "user@company.com"
```

#### 3. Examine Role and ClusterRole Definitions
```bash
# Check what roles exist in the namespace
kubectl get roles -n development

# Examine specific role permissions
kubectl describe role <role-name> -n development

# Check cluster roles that might be bound
kubectl get clusterroles
kubectl describe clusterrole <clusterrole-name>
```

#### 4. Verify User Identity and Groups
```bash
# Check if user is properly authenticated
kubectl config view --minify  # Current context
kubectl config get-users      # Available users

# For service accounts, check if they exist
kubectl get serviceaccounts -n development
```

#### 5. Common Issues and Fixes
```bash
# Issue 1: Missing RoleBinding
# Solution: Create appropriate binding
kubectl create rolebinding developer-pod-creator \
  --role=pod-creator \
  --user=user@company.com \
  --namespace=development

# Issue 2: Role doesn't have required permissions
# Solution: Update the role
kubectl patch role developer -n development --type='json' -p='[{"op": "add", "path": "/rules/-", "value": {"apiGroups": [""], "resources": ["pods"], "verbs": ["create"]}}]'

# Issue 3: User not in correct group
# Solution: Bind to group instead
kubectl create rolebinding developers-pod-creator \
  --role=pod-creator \
  --group=developers \
  --namespace=development
```

### Debugging Script
```bash
#!/bin/bash
# rbac-debug.sh

USER=$1
NAMESPACE=$2

echo "=== Debugging RBAC for user: $USER in namespace: $NAMESPACE ==="

echo -e "\n1. Testing basic pod operations:"
kubectl auth can-i create pods --as=$USER --namespace=$NAMESPACE
kubectl auth can-i get pods --as=$USER --namespace=$NAMESPACE
kubectl auth can-i delete pods --as=$USER --namespace=$NAMESPACE

echo -e "\n2. Checking all permissions:"
kubectl auth can-i --list --as=$USER --namespace=$NAMESPACE

echo -e "\n3. Checking rolebindings in namespace:"
kubectl get rolebindings -n $NAMESPACE -o wide

echo -e "\n4. Checking clusterrolebindings:"
kubectl get clusterrolebindings -o wide | grep -E "(NAME|$USER)"

echo -e "\n5. Detailed binding information:"
for binding in $(kubectl get rolebindings -n $NAMESPACE -o name); do
  echo "=== $binding ==="
  kubectl get $binding -n $NAMESPACE -o yaml | grep -E "(subjects|roleRef)"
done
```

---

## Scenario 8: Custom Resource Definitions

**Question**: Your team has created Custom Resource Definitions (CRDs). How would you grant developers permission to manage these custom resources?

**Answer**:

### RBAC for Custom Resources
```yaml
# Custom Resource Definition
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
---
# RBAC for custom resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-manager
rules:
- apiGroups: ["example.com"]  # Custom API group
  resources: ["applications"]  # Custom resource
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["example.com"]
  resources: ["applications/status"]  # Subresource
  verbs: ["get", "update", "patch"]
---
# Binding for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-application-manager
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: application-manager
  apiGroup: rbac.authorization.k8s.io
```

### Fine-grained CRD Permissions
```yaml
# Read-only access to custom resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-viewer
rules:
- apiGroups: ["example.com"]
  resources: ["applications"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["example.com"]
  resources: ["applications/status"]
  verbs: ["get"]
---
# Admin access with additional privileges
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: application-admin
rules:
- apiGroups: ["example.com"]
  resources: ["applications"]
  verbs: ["*"]  # All verbs
- apiGroups: ["example.com"]
  resources: ["applications/status", "applications/finalizers"]
  verbs: ["*"]
- apiGroups: [""]  # Need access to related resources
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update"]
```

---

## Scenario 9: Multi-team Cluster

**Question**: You manage a Kubernetes cluster used by multiple teams. How would you design an RBAC strategy that provides isolation while allowing necessary cross-team collaboration?

**Answer**:

### Multi-team RBAC Architecture

#### 1. Base Namespace Structure
```bash
# Create namespaces for each team
kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace team-database
kubectl create namespace shared-services
```

#### 2. Team-specific Roles
```yaml
# team-base-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: team-base-permissions
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["*"]
---
# Apply to each team namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-team-binding
  namespace: team-frontend
subjects:
- kind: Group
  name: "team-frontend"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: team-base-permissions
  apiGroup: rbac.authorization.k8s.io
```

#### 3. Cross-team Collaboration Roles
```yaml
# cross-team-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cross-team-reader
rules:
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list"]
  # No access to secrets or pods
---
# Allow frontend to read backend services
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: frontend-read-backend
  namespace: team-backend
subjects:
- kind: Group
  name: "team-frontend"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cross-team-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 4. Shared Services Access
```yaml
# shared-services-monitoring.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: shared-services
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
---
# Allow all teams to read monitoring data
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: all-teams-monitoring
  namespace: shared-services
subjects:
- kind: Group
  name: "team-frontend"
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: "team-backend"
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: "team-database"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Scenario 10: Audit and Compliance

**Question**: For compliance requirements, you need to audit all RBAC changes and regularly review permissions. How would you implement this?

**Answer**:

### 1. Enable Kubernetes Audit Logging
```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  namespaces: ["kube-system"]
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["clusterroles", "clusterrolebindings"]
- level: Request
  users: ["system:serviceaccount:kube-system:cluster-admin"]
```

### 2. Regular RBAC Reviews
```bash
#!/bin/bash
# rbac-audit-script.sh

echo "=== RBAC Audit Report ==="
echo "Generated: $(date)"

echo -e "\n1. ClusterRoles with Wildcard Verbs:"
kubectl get clusterroles -o yaml | grep -B 10 -A 10 "verbs:.*\\*"

echo -e "\n2. ClusterRoles with Wildcard Resources:"
kubectl get clusterroles -o yaml | grep -B 10 -A 10 "resources:.*\\*"

echo -e "\n3. ClusterRoleBindings to Default Service Accounts:"
kubectl get clusterrolebindings -o yaml | grep -B 5 -A 5 "system:serviceaccount"

echo -e "\n4. High-Privilege ClusterRoleBindings:"
kubectl describe clusterrole cluster-admin | grep -A 20 "Rules"
kubectl get clusterrolebindings -o wide | grep cluster-admin

echo -e "\n5. Service Accounts with Cluster-wide Access:"
kubectl get clusterrolebindings -o yaml | grep -B 10 -A 10 "kind: ServiceAccount"
```

### 3. RBAC Validation Tools
```yaml
# rbac-lookup installation and usage
# Install rbac-lookup
kubectl krew install rbac-lookup

# Check permissions for a user
kubectl rbac-lookup user@company.com

# Check all service accounts with cluster-admin
kubectl rbac-lookup --grep cluster-admin

# Audit tool using kubectl
apiVersion: v1
kind: Pod
metadata:
  name: rbac-auditor
spec:
  containers:
  - name: auditor
    image: bitnami/kubectl:latest
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "=== Cluster-wide RBAC Audit ==="
      kubectl get clusterrolebindings -o wide
      echo ""
      echo "=== High Privilege Bindings ==="
      kubectl get clusterrolebindings -o yaml | grep -B 20 -A 5 "cluster-admin"
      echo ""
      echo "=== Service Accounts with Cluster Roles ==="
      kubectl get clusterrolebindings -o yaml | grep -B 10 -A 10 "ServiceAccount"
  restartPolicy: Never
```

### 4. Automated RBAC Review with CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rbac-audit
  namespace: kube-system
spec:
  schedule: "0 2 * * 1"  # Every Monday at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: rbac-auditor
          containers:
          - name: audit
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Generate RBAC report
              kubectl get clusterroles,clusterrolebindings,roles,rolebindings --all-namespaces -o yaml > /tmp/rbac-state-$(date +%Y%m%d).yaml
              
              # Check for risky permissions
              kubectl get clusterroles -o yaml | grep -B 5 -A 5 "\\*" > /tmp/wildcard-permissions-$(date +%Y%m%d).txt
              
              # Send report (implementation depends on your setup)
              echo "RBAC audit completed at $(date)"
          restartPolicy: OnFailure
---
# Service account for the audit job
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbac-auditor
  namespace: kube-system
---
# Minimal permissions for audit
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rbac-auditor
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rbac-auditor-binding
subjects:
- kind: ServiceAccount
  name: rbac-auditor
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: rbac-auditor
  apiGroup: rbac.authorization.k8s.io
```


## License
This project is licensed under the MIT License.