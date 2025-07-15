# Kubernetes Service: Scenario-Based Interview Questions  

A collection of scenario-based interview questions focused on Kubernets Service.  

---

## Table of Contents  
1. [Service Fundamentals](#1-service-fundamentals)  
2. [ClusterIP Internal Communication](#2-clusterip-internal-communication)  
3. [NodePort External Access Issue](#3-nodeport-external-access-issue)  
4. [LoadBalancer Cloud Integration](#4-loadbalancer-cloud-integration)  
5. [ExternalName DNS Mapping](#5-externalname-dns-mapping)  
6. [Advanced Service Patterns](#6-advanced-service-patterns)  
7. [Interview Strategies](#7-interview-strategies)  

---

## 1. Service Fundamentals  

**Question**:  
*"Explain how a Kubernetes Service maintains stable IP addressing despite Pod restarts."*  

**Key Concepts**:  
- **Virtual IP Abstraction**:  
  - Services provide a stable VIP (ClusterIP) that persists across Pod changes  
  - Underlying `kube-proxy` manages iptables/IPVS rules to route traffic  

- **Endpoint Synchronization**:  
  ```sh
  kubectl get endpoints <service-name>  # Shows current Pod IPs
  ```  
- **Core Service Types**:  
  | Type | Default Port Range | Use Case |  
  |------|-------------------|----------|  
  | ClusterIP | 10.96.0.0/12 | Intra-cluster communication |  
  | NodePort | 30000-32767 | Development/testing exposure |  
  | LoadBalancer | Varies | Production external access |  
  | ExternalName | N/A | CNAME-like DNS mapping |  

---

## 2. ClusterIP Internal Communication  

**Scenario**:  
*"Your backend microservices cannot communicate via their ClusterIP Services. Diagnose the issue."*  

**Debugging Flow**:  
1. **Verify Service-Pod Binding**:  
   ```sh
   kubectl describe svc backend-service | grep -A 5 Selector
   kubectl get pods -l app=backend --show-labels
   ```  
2. **Check Network Policies**:  
   ```sh
   kubectl get networkpolicy -n <namespace>
   ```  
3. **Test DNS Resolution**:  
   ```sh
   kubectl exec -it test-pod -- nslookup backend-service.<namespace>
   ```  
4. **Common Pitfalls**:  
   - Namespace mismatches  
   - NetworkPolicy blocking traffic  
   - CoreDNS misconfiguration  

---

## 3. NodePort External Access Issue  

**Scenario**:  
*"Users report timeout errors when accessing your NodePort Service (port 31000). Resolve this."*  

**Resolution Steps**:  
1. **Validate NodePort Allocation**:  
   ```sh
   kubectl get svc -o jsonpath='{range .items[*]}{.spec.ports[0].nodePort}{"\n"}{end}'
   ```  
2. **Check Node Firewall Rules**:  
   ```sh
   sudo iptables -L -n -t nat | grep 31000
   ```  
3. **Test Direct Pod Access**:  
   ```sh
   kubectl port-forward pod/<pod-name> 8080:80  # Bypass Service
   ```  
4. **Configuration Example**:  
   ```yaml
   spec:
     ports:
       - nodePort: 31000
         port: 80
         protocol: TCP
     type: NodePort
   ```

---

## 4. LoadBalancer Cloud Integration  

**Scenario**:  
*"Your LoadBalancer Service on AWS isn't receiving traffic. Cloud logs show health check failures."*  

**Cloud-Specific Solutions**:  
1. **Annotate for AWS ALB**:  
   ```yaml
   metadata:
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /healthz
   ```  
2. **Verify Target Groups**:  
   - Confirm Pod readiness probes match LB health checks  
   ```sh
   kubectl describe pod <pod-name> | grep -A 5 Readiness
   ```  
3. **Troubleshooting Matrix**:  
   | Cloud | Diagnostic Command |  
   |-------|--------------------|  
   | AWS | `aws elbv2 describe-target-health` |  
   | GCP | `gcloud compute backend-services get-health` |  
   | Azure | `az network lb probe list` |  

---

## 5. ExternalName DNS Mapping  

**Scenario**:  
*"Migrate a legacy system to Kubernetes while maintaining existing DNS references."*  

**Implementation**:  
1. **Service Definition**:  
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: legacy-system
   spec:
     type: ExternalName
     externalName: old-system.example.com
   ```  
2. **DNS Validation**:  
   ```sh
   kubectl run -it --rm test-nslookup --image=busybox -- \
     nslookup legacy-system.default.svc.cluster.local
   ```  
3. **Migration Phases**:  
   - Phase 1: ExternalName Service as proxy  
   - Phase 2: Deploy Kubernetes-native replacement  
   - Phase 3: Update Service to ClusterIP type  

---

## 6. Advanced Service Patterns  

**Question**:  
*"How would you implement canary traffic splitting between two Services?"*  

**Solutions**:  
1. **Service Mesh Approach** (Istio):  
   ```yaml
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   spec:
     hosts: ["my-app"]
     http:
     - route:
       - destination:
           host: stable-svc
         weight: 90
       - destination:
           host: canary-svc
         weight: 10
   ```  
2. **Ingress-Based Solution** (Nginx):  
   ```yaml
   annotations:
     nginx.ingress.kubernetes.io/canary: "true"
     nginx.ingress.kubernetes.io/canary-weight: "10"
   ```  

---

## 7. Interview Strategies  

**STAR Method Examples**:  
- **Situation**: "Our e-commerce platform needed zero-downtime payment processing upgrades..."  
- **Task**: "Required migrating from NodePort to LoadBalancer Services..."  
- **Action**: "Implemented pre-flight checks using `kubectl get endpoints`..."  
- **Result**: "Achieved 99.95% uptime during quarterly upgrades..."  

**Must-Know Commands**:  
```sh
# Watch Service endpoint changes
kubectl get endpoints <service> -w

# Diagnose kube-proxy issues
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

---
