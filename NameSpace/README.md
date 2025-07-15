# Kubernetes Namespace Scenario-Based Interview Questions

A collection of scenario-based interview about Kubernetes namespaces.

---

## 1. Multi-Tenancy Scenario  
**Question:**  
Your company wants to deploy applications for different departments (Finance, HR, Marketing) on the same cluster. How would you use namespaces to implement this securely?  

**Key Points to Address:**  
- Create separate namespaces for each department (e.g., `finance-ns`, `hr-ns`, `marketing-ns`).  
- Implement RBAC with RoleBindings to restrict department access only to their namespace.  
- Configure NetworkPolicies to control inter-namespace communication.  
- Consider ResourceQuotas to prevent one department from consuming all cluster resources.  
- Use labels for cost allocation and monitoring per department.  

---

## 2. Namespace Resource Management  
**Question:**  
Your team reports that their pods in the `dev` namespace keep getting evicted. What could be the issue and how would you troubleshoot?  

**Approach:**  
- Check if ResourceQuotas are set on the `dev` namespace limiting CPU/memory.  
- Examine LimitRanges that might be restricting pod resources.  
- Investigate cluster-level resource pressure with `kubectl top nodes`.  
- Review pod eviction messages with `kubectl get events -n dev`.  
- Consider whether the namespace has appropriate priority classes assigned.  

---

## 3. Namespace Isolation Challenge  
**Question:**  
Developers in the `staging` namespace complain they can't access a service running in `production`. How would you enable this access while maintaining security?  

**Solution Components:**  
- Create a NetworkPolicy allowing specific ingress from `staging` to `production`.  
- Consider using fully qualified domain names (e.g., `service.production.svc.cluster.local`).  
- Implement service exports/imports if using a service mesh like Istio.  
- Set up proper RBAC for cross-namespace service access.  
- Document the dependencies between namespaces.  

---

## 4. Namespace Lifecycle Management  
**Question:**  
Your CI/CD pipeline needs to create temporary namespaces for each feature branch. How would you automate this safely?  

**Implementation Strategy:**  
- Use Kubernetes operators or controllers to manage namespace lifecycle.  
- Implement namespace templates with baseline configurations (RBAC, quotas, etc.).  
- Set up automatic cleanup with TTL annotations or external controllers.  
- Integrate with your CI system to trigger namespace creation/deletion.  
- Include monitoring to detect and alert on orphaned namespaces.  

---

## 5. Namespace Migration Scenario  
**Question:**  
You need to migrate all resources from an old namespace to a new one. How would you approach this with minimal downtime?  

**Migration Steps:**  
1. Use `kubectl get all -n old-ns -o yaml > resources.yaml`.  
2. Modify the namespace field in the YAML files.  
3. Consider using tools like Velero for stateful workloads.  
4. For services, implement DNS aliases during transition.  
5. Test with canary deployments before full migration.  
6. Update all references in CI/CD, monitoring, and logging systems.  

---

## 6. Namespace Security Incident  
**Question:**  
You suspect a compromised pod in the `monitoring` namespace is trying to access resources in other namespaces. How would you investigate and contain this?  

**Incident Response:**  
- Immediately apply NetworkPolicy to restrict egress from the `monitoring` namespace.  
- Check audit logs for suspicious cross-namespace API calls.  
- Review RBAC bindings in the `monitoring` namespace for excessive permissions.  
- Consider pod security policies/admission controllers to prevent privilege escalation.  
- Use `kubectl audit` to trace namespace-crossing activities.  

---

## 7. Namespace Design Best Practices  
**Question:**  
When designing a namespace strategy for a new project, what factors would you consider?  

**Design Considerations:**  
- Purpose-based grouping (by team, environment, application, etc.).  
- Security and isolation requirements.  
- Resource management needs (quotas, limits).  
- Operational overhead vs. isolation benefits.  
- Naming conventions and labeling strategy.  
- Lifecycle management approach.  
- Monitoring and cost allocation requirements.  

