# Kubernetes Replication Controller and ReplicaSet Scenario-Based Interview Questions

A collection of scenario-based interview questions focused on Kubernetes Replication Controller and ReplicaSet.

---

## Table of Contents  
1. [Replication Controller vs. ReplicaSet](#1-replication-controller-vs-replicaset)  
2. [Scaling Application Scenario](#2-scaling-application-scenario)  
3. [Pod Failure Recovery Scenario](#3-pod-failure-recovery-scenario)  
4. [Rolling Update Challenge](#4-rolling-update-challenge)  
5. [Resource Management Issue](#5-resource-management-issue)  
6. [ReplicaSet Design Best Practices](#6-replicaset-design-best-practices)  

---

## 1. Replication Controller vs. ReplicaSet  
**Question:**  
What are the key differences between a Replication Controller and a ReplicaSet, and when would you use one over the other?  

**Key Points to Address:**  
- **Replication Controller**: Older API object, uses equality-based selectors.  
- **ReplicaSet**: Newer and more flexible, supports set-based selectors (e.g., `in`, `notin`, `exists`).  
- Use **ReplicaSet** for more advanced selector requirements.  
- **Replication Controller** is still supported but considered legacy.  
- Both ensure a specified number of pod replicas are running.  

---

## 2. Scaling Application Scenario  
**Question:**  
Your application is experiencing high traffic, and you need to scale up the number of pods from 3 to 10. How would you achieve this using a ReplicaSet?  

**Steps to Follow:**  
1. Edit the ReplicaSet definition: `kubectl edit rs <replicaset-name>`.  
2. Update the `replicas` field from 3 to 10.  
3. Save and exit the editor.  
4. Verify scaling with `kubectl get pods`.  
5. Consider using Horizontal Pod Autoscaler (HPA) for automatic scaling in the future.  

---

## 3. Pod Failure Recovery Scenario  
**Question:**  
One of the pods managed by a ReplicaSet has crashed. How does the ReplicaSet handle this situation?  

**Explanation:**  
- ReplicaSet continuously monitors the number of running pods.  
- If a pod crashes, the ReplicaSet detects the discrepancy between desired and actual replicas.  
- It automatically creates a new pod to replace the crashed one.  
- This ensures the desired state (number of replicas) is maintained.  

---

## 4. Rolling Update Challenge  
**Question:**  
You need to update the image of your application pods managed by a ReplicaSet. How would you perform a rolling update without downtime?  

**Approach:**  
1. Update the ReplicaSet's pod template with the new image.  
2. Delete old pods one by one: `kubectl delete pod <pod-name>`.  
3. The ReplicaSet will create new pods with the updated image.  
4. Alternatively, use a Deployment object for better rolling update management.  

---

## 5. Resource Management Issue  
**Question:**  
Your ReplicaSet is creating too many pods, causing resource exhaustion on the cluster. How would you troubleshoot and resolve this?  

**Troubleshooting Steps:**  
- Check the `replicas` field in the ReplicaSet definition.  
- Verify if Horizontal Pod Autoscaler (HPA) is misconfigured.  
- Examine resource requests and limits for the pods.  
- Use `kubectl describe rs <replicaset-name>` to inspect events.  
- Adjust the `replicas` count or implement ResourceQuotas.  

---

## 6. ReplicaSet Design Best Practices  
**Question:**  
When designing a ReplicaSet for a production application, what best practices would you follow?  

**Best Practices:**  
- Use **Deployments** instead of directly managing ReplicaSets for better update strategies.  
- Set appropriate resource requests and limits for pods.  
- Use labels and selectors effectively for managing pod groups.  
- Implement Horizontal Pod Autoscaler (HPA) for dynamic scaling.  
- Monitor pod health and ReplicaSet status with tools like Prometheus.  

