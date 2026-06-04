# Kubernetes Reference Cheatsheet

> Quick-reference guide for Kubernetes core resources, scheduling, networking, troubleshooting, and autoscaling

---

## 1. Core Scheduling & Pod Isolation

- **Taints & Tolerations**:
  - Taints are applied to nodes (e.g. `gpu=true:NoSchedule`). Pods must have a matching toleration to be scheduled on that node.
- **Node Affinity**:
  - `requiredDuringSchedulingIgnoredDuringExecution`: Hard constraint (must match labels to schedule).
  - `preferredDuringSchedulingIgnoredDuringExecution`: Soft constraint (scheduler tries to match).
- **Requests vs Limits**:
  - **Requests**: Minimum resources guaranteed to the container (used for scheduling decisions).
  - **Limits**: Maximum resources the container can consume. Exceeding CPU triggers throttling; exceeding memory triggers **OOMKilled** termination.

---

## 2. Pod Networking & Traffic Flow

- **ClusterIP**: Default service type. Exposes the service on an internal IP address reachable only within the cluster.
- **NodePort**: Exposes the service on a static port on each node's IP. External traffic hits node_ip:nodeport to route inside.
- **LoadBalancer**: Provisions an external cloud load balancer (e.g., AWS ALB/NLB) routing traffic to NodePort/ClusterIP.
- **CoreDNS**: In-cluster DNS resolver translating service names to IP addresses (e.g. `http://payment-service.production.svc.cluster.local`).

---

## 3. Trouble-shooting Checklist

- **Identify CrashReason**:
  - Check pod description: `kubectl describe pod <pod_name>`
  - Look for status: `CrashLoopBackOff`, `OOMKilled`, or `ImagePullBackOff`.
- **View Container Logs**:
  - Standard logs: `kubectl logs <pod_name> -n <namespace> --tail=100`
  - Previous failed container logs: `kubectl logs <pod_name> -p`
- **Execute shell inside container**:
  - `kubectl exec -it <pod_name> -n <namespace> -- /bin/sh`
