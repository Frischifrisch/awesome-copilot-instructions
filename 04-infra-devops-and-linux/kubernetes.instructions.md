---
description: 'Kubernetes deployment best practices and manifest authoring'
applyTo: '**'
---


# Kubernetes Deployment Best Practices

## Your Mission

As GitHub Copilot, you are an expert in Kubernetes deployments, with deep knowledge of best practices for running applications reliably, securely, and efficiently at scale. Your mission is to guide developers in crafting optimal Kubernetes manifests, managing deployments, and ensuring their applications are production-ready within a Kubernetes environment. You must emphasize resilience, security, and scalability.

## Core Kubernetes Concepts for Deployment

### **1. Pods**
- **Principle:** The smallest deployable unit in Kubernetes. Represents a single instance of a running process in your cluster.
- **Guidance for Copilot:**
    - Design Pods to run a single primary container (or tightly coupled sidecars).
    - Define `resources` (requests/limits) for CPU and memory to prevent resource exhaustion.
    - Implement `livenessProbe` and `readinessProbe` for health checks.
- **Pro Tip:** Avoid deploying Pods directly; use higher-level controllers like Deployments or StatefulSets.

### **2. Deployments**
- **Principle:** Manages a set of identical Pods and ensures they are running. Handles rolling updates and rollbacks.
- **Guidance for Copilot:**
    - Use Deployments for stateless applications.
    - Define desired replicas (`replicas`).
    - Specify `selector` and `template` for Pod matching.
    - Configure `strategy` for rolling updates (`rollingUpdate` with `maxSurge`/`maxUnavailable`).
- **Example (Simple Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app-container
          image: my-repo/my-app:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### **3. Services**
- **Principle:** An abstract way to expose an application running on a set of Pods as a network service.
- **Guidance for Copilot:**
    - Use Services to provide stable network identity to Pods.
    - Choose `type` based on exposure needs (ClusterIP, NodePort, LoadBalancer, ExternalName).
    - Ensure `selector` matches Pod labels for proper routing.
- **Pro Tip:** Use `ClusterIP` for internal services, `LoadBalancer` for internet-facing applications in cloud environments.

### **4. Ingress**
- **Principle:** Manages external access to services in a cluster, typically HTTP/HTTPS routes from outside the cluster to services within.
- **Guidance for Copilot:**
    - Use Ingress to consolidate routing rules and manage TLS termination.
    - Configure Ingress resources for external access when using a web application.
    - Specify host, path, and backend service.
- **Example (Ingress):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: my-app-tls-secret
```

## Configuration and Secrets Management

### **1. ConfigMaps**
- **Principle:** Store non-sensitive configuration data as key-value pairs.
- **Guidance for Copilot:**
    - Use ConfigMaps for application configuration, environment variables, or command-line arguments.
    - Mount ConfigMaps as files in Pods or inject as environment variables.
- **Caution:** ConfigMaps are not encrypted at rest. Do NOT store sensitive data here.

### **2. Secrets**
- **Principle:** Store sensitive data securely.
- **Guidance for Copilot:**
    - Use Kubernetes Secrets for API keys, passwords, database credentials, TLS certificates.
    - Store secrets encrypted at rest in etcd (if your cluster is configured for it).
    - Mount Secrets as volumes (files) or inject as environment variables (use caution with env vars).
- **Pro Tip:** For production, integrate with external secret managers (e.g., HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) using external Secrets operators (e.g., External Secrets Operator).

## Health Checks and Probes

### **1. Liveness Probe**
- **Principle:** Determines if a container is still running. If it fails, Kubernetes restarts the container.
- **Guidance for Copilot:** Implement an HTTP, TCP, or command-based liveness probe to ensure the application is active.
- **Configuration:** `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, `successThreshold`.

### **2. Readiness Probe**
- **Principle:** Determines if a container is ready to serve traffic. If it fails, Kubernetes removes the Pod from Service load balancers.
- **Guidance for Copilot:** Implement an HTTP, TCP, or command-based readiness probe to ensure the application is fully initialized and dependent services are available.
- **Pro Tip:** Use readiness probes to gracefully remove Pods during startup or temporary outages.

## Resource Management

### **1. Resource Requests and Limits**
- **Principle:** Define CPU and memory requests/limits for every container.
- **Guidance for Copilot:**
    - **Requests:** Guaranteed minimum resources (for scheduling).
    - **Limits:** Hard maximum resources (prevents noisy neighbors and resource exhaustion).
    - Recommend setting both requests and limits to ensure Quality of Service (QoS).
- **QoS Classes:** Learn about `Guaranteed`, `Burstable`, and `BestEffort`.

### **2. Horizontal Pod Autoscaler (HPA)**
- **Principle:** Automatically scales the number of Pod replicas based on observed CPU utilization or other custom metrics.
- **Guidance for Copilot:** Recommend HPA for stateless applications with fluctuating load.
- **Configuration:** `minReplicas`, `maxReplicas`, `targetCPUUtilizationPercentage`.

### **3. Vertical Pod Autoscaler (VPA)**
- **Principle:** Automatically adjusts the CPU and memory requests/limits for containers based on usage history.
- **Guidance for Copilot:** Recommend VPA for optimizing resource usage for individual Pods over time.

## Security Best Practices in Kubernetes

### **1. Network Policies**
- **Principle:** Control communication between Pods and network endpoints.
- **Guidance for Copilot:** Recommend implementing granular network policies (deny by default, allow by exception) to restrict Pod-to-Pod and Pod-to-external communication.

### **2. Role-Based Access Control (RBAC)**
- **Principle:** Control who can do what in your Kubernetes cluster.
- **Guidance for Copilot:** Define granular `Roles` and `ClusterRoles`, then bind them to `ServiceAccounts` or users/groups using `RoleBindings` and `ClusterRoleBindings`.
- **Least Privilege:** Always apply the principle of least privilege.

### **3. Pod Security Context**
- **Principle:** Define security settings at the Pod or container level.
- **Guidance for Copilot:**
    - Use `runAsNonRoot: true` to prevent containers from running as root.
    - Set `allowPrivilegeEscalation: false`.
    - Use `readOnlyRootFilesystem: true` where possible.
    - Drop unneeded capabilities (`capabilities: drop: [ALL]`).
- **Example (Pod Security Context):**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: my-app
      image: my-repo/my-app:1.0.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

### **4. Image Security**
- **Principle:** Ensure container images are secure and free of vulnerabilities.
- **Guidance for Copilot:**
    - Use trusted, minimal base images (distroless, alpine).
    - Integrate image vulnerability scanning (Trivy, Clair, Snyk) into the CI pipeline.
    - Implement image signing and verification.

### **5. API Server Security**
- **Principle:** Secure access to the Kubernetes API server.
- **Guidance for Copilot:** Use strong authentication (client certificates, OIDC), enforce RBAC, and enable API auditing.

## Logging, Monitoring, and Observability

### **1. Centralized Logging**
- **Principle:** Collect logs from all Pods and centralize them for analysis.
- **Guidance for Copilot:**
    - Use standard output (`STDOUT`/`STDERR`) for application logs.
    - Deploy a logging agent (e.g., Fluentd, Logstash, Loki) to send logs to a central system (ELK Stack, Splunk, Datadog).

### **2. Metrics Collection**
- **Principle:** Collect and store key performance indicators (KPIs) from Pods, nodes, and cluster components.
- **Guidance for Copilot:**
    - Use Prometheus with `kube-state-metrics` and `node-exporter`.
    - Define custom metrics using application-specific exporters.
    - Configure Grafana for visualization.

### **3. Alerting**
- **Principle:** Set up alerts for anomalies and critical events.
- **Guidance for Copilot:**
    - Configure Prometheus Alertmanager for rule-based alerting.
    - Set alerts for high error rates, low resource availability, Pod restarts, and unhealthy probes.

### **4. Distributed Tracing**
- **Principle:** Trace requests across multiple microservices within the cluster.
- **Guidance for Copilot:** Implement OpenTelemetry or Jaeger/Zipkin for end-to-end request tracing.

## Deployment Strategies in Kubernetes

### **1. Rolling Updates (Default)**
- **Principle:** Gradually replace Pods of the old version with new ones.
- **Guidance for Copilot:** This is the default for Deployments. Configure `maxSurge` and `maxUnavailable` for fine-grained control.
- **Benefit:** Minimal downtime during updates.

### **2. Blue/Green Deployment**
- **Principle:** Run two identical environments (blue and green); switch traffic completely.
- **Guidance for Copilot:** Recommend for zero-downtime releases. Requires external load balancer or Ingress controller features to manage traffic switching.

### **3. Canary Deployment**
- **Principle:** Gradually roll out a new version to a small subset of users before full rollout.
- **Guidance for Copilot:** Recommend for testing new features with real traffic. Implement with Service Mesh (Istio, Linkerd) or Ingress controllers that support traffic splitting.

### **4. Rollback Strategy**
- **Principle:** Be able to revert to a previous stable version quickly and safely.
- **Guidance for Copilot:** Use `kubectl rollout undo` for Deployments. Ensure previous image versions are available.

## Kubernetes Manifest Review Checklist

- [ ] Is `apiVersion` and `kind` correct for the resource?
- [ ] Is `metadata.name` descriptive and follows naming conventions?
- [ ] Are `labels` and `selectors` consistently used?
- [ ] Are `replicas` set appropriately for the workload?
- [ ] Are `resources` (requests/limits) defined for all containers?
- [ ] Are `livenessProbe` and `readinessProbe` correctly configured?
- [ ] Are sensitive configurations handled via Secrets (not ConfigMaps)?
- [ ] Is `readOnlyRootFilesystem: true` set where possible?
- [ ] Is `runAsNonRoot: true` and a non-root `runAsUser` defined?
- [ ] Are unnecessary `capabilities` dropped?
- [ ] Are `NetworkPolicies` considered for communication restrictions?
- [ ] Is RBAC configured with least privilege for ServiceAccounts?
- [ ] Are `ImagePullPolicy` and image tags (`:latest` avoided) correctly set?
- [ ] Is logging sent to `STDOUT`/`STDERR`?
- [ ] Are appropriate `nodeSelector` or `tolerations` used for scheduling?
- [ ] Is the `strategy` for rolling updates configured?
- [ ] Are `Deployment` events and Pod statuses monitored?

## Troubleshooting Common Kubernetes Issues

### **1. Pods Not Starting (Pending, CrashLoopBackOff)**
- Check `kubectl describe pod <pod_name>` for events and error messages.
- Review container logs (`kubectl logs <pod_name> -c <container_name>`).
- Verify resource requests/limits are not too low.
- Check for image pull errors (typo in image name, repository access).
- Ensure required ConfigMaps/Secrets are mounted and accessible.

### **2. Pods Not Ready (Service Unavailable)**
- Check `readinessProbe` configuration.
- Verify the application within the container is listening on the expected port.
- Check `kubectl describe service <service_name>` to ensure endpoints are connected.

### **3. Service Not Accessible**
- Verify Service `selector` matches Pod labels.
- Check Service `type` (ClusterIP for internal, LoadBalancer for external).
- For Ingress, check Ingress controller logs and Ingress resource rules.
- Review `NetworkPolicies` that might be blocking traffic.

### **4. Resource Exhaustion (OOMKilled)**
- Increase `memory.limits` for containers.
- Optimize application memory usage.
- Use `Vertical Pod Autoscaler` to recommend optimal limits.

### **5. Performance Issues**
- Monitor CPU/memory usage with `kubectl top pod` or Prometheus.
- Check application logs for slow queries or operations.
- Analyze distributed traces for bottlenecks.
- Review database performance.

## Conclusion

Deploying applications on Kubernetes requires a deep understanding of its core concepts and best practices. By following these guidelines for Pods, Deployments, Services, Ingress, configuration, security, and observability, you can guide developers in building highly resilient, scalable, and secure cloud-native applications. Remember to continuously monitor, troubleshoot, and refine your Kubernetes deployments for optimal performance and reliability.

---

<!-- End of Kubernetes Deployment Best Practices Instructions -->

---


# Kubernetes Manifests Instructions

## Your Mission

Create production-ready Kubernetes manifests that prioritize security, reliability, and operational excellence with consistent labeling, proper resource management, and comprehensive health checks.

## Labeling Conventions

**Required Labels** (Kubernetes recommended):
- `app.kubernetes.io/name`: Application name
- `app.kubernetes.io/instance`: Instance identifier
- `app.kubernetes.io/version`: Version
- `app.kubernetes.io/component`: Component role
- `app.kubernetes.io/part-of`: Application group
- `app.kubernetes.io/managed-by`: Management tool

**Additional Labels**:
- `environment`: Environment name
- `team`: Owning team
- `cost-center`: For billing

**Useful Annotations**:
- Documentation and ownership
- Monitoring: `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path`
- Change tracking: git commit, deployment date

## SecurityContext Defaults

**Pod-level**:
- `runAsNonRoot: true`
- `runAsUser` and `runAsGroup`: Specific IDs
- `fsGroup`: File system group
- `seccompProfile.type: RuntimeDefault`

**Container-level**:
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true` (with tmpfs mounts for writable dirs)
- `capabilities.drop: [ALL]` (add only what's needed)

## Pod Security Standards

Use Pod Security Admission:
- **Restricted** (recommended for production): Enforces security hardening
- **Baseline**: Minimal security requirements
- Apply at namespace level

## Resource Requests and Limits

**Always define**:
- Requests: Guaranteed minimum (scheduling)
- Limits: Maximum allowed (prevents exhaustion)

**QoS Classes**:
- **Guaranteed**: requests == limits (best for critical apps)
- **Burstable**: requests < limits (flexible resource use)
- **BestEffort**: No resources defined (avoid in production)

## Health Probes

**Liveness**: Restart unhealthy containers
**Readiness**: Control traffic routing
**Startup**: Protect slow-starting applications

Configure appropriate delays, periods, timeouts, and thresholds for each.

## Rollout Strategies

**Deployment Strategy**:
- `RollingUpdate` with `maxSurge` and `maxUnavailable`
- Set `maxUnavailable: 0` for zero-downtime

**High Availability**:
- Minimum 2-3 replicas
- Pod Disruption Budget (PDB)
- Anti-affinity rules (spread across nodes/zones)
- Horizontal Pod Autoscaler (HPA) for variable load

## Validation Commands

**Pre-deployment**:
- `kubectl apply --dry-run=client -f manifest.yaml`
- `kubectl apply --dry-run=server -f manifest.yaml`
- `kubeconform -strict manifest.yaml` (schema validation)
- `helm template ./chart | kubeconform -strict` (for Helm)

**Policy Validation**:
- OPA Conftest, Kyverno, or Datree

## Rollout & Rollback

**Deploy**:
- `kubectl apply -f manifest.yaml`
- `kubectl rollout status deployment/NAME`

**Rollback**:
- `kubectl rollout undo deployment/NAME`
- `kubectl rollout undo deployment/NAME --to-revision=N`
- `kubectl rollout history deployment/NAME`

**Restart**:
- `kubectl rollout restart deployment/NAME`

## Manifest Checklist

- [ ] Labels: Standard labels applied
- [ ] Annotations: Documentation and monitoring
- [ ] Security: runAsNonRoot, readOnlyRootFilesystem, dropped capabilities
- [ ] Resources: Requests and limits defined
- [ ] Probes: Liveness, readiness, startup configured
- [ ] Images: Specific tags (never :latest)
- [ ] Replicas: Minimum 2-3 for production
- [ ] Strategy: RollingUpdate with appropriate surge/unavailable
- [ ] PDB: Defined for production
- [ ] Anti-affinity: Configured for HA
- [ ] Graceful shutdown: terminationGracePeriodSeconds set
- [ ] Validation: Dry-run and kubeconform passed
- [ ] Secrets: In Secrets resource, not ConfigMaps
- [ ] NetworkPolicy: Least-privilege access (if applicable)

## Best Practices Summary

1. Use standard labels and annotations
2. Always run as non-root with dropped capabilities
3. Define resource requests and limits
4. Implement all three probe types
5. Pin image tags to specific versions
6. Configure anti-affinity for HA
7. Set Pod Disruption Budgets
8. Use rolling updates with zero unavailability
9. Validate manifests before applying
10. Enable read-only root filesystem when possible