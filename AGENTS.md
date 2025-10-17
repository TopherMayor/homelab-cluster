# K3s GitOps Repository - Agent Guidelines

## Build/Lint/Test Commands
This is a k3s cluster GitOps repository using Flux CD. No traditional build/test commands exist.

### Validation Commands
```bash
# Validate Kubernetes manifests against k3s
kubectl apply --dry-run=client -f clusters/pi-cluster/apps/

# Validate Flux Kustomization
flux validate kustomization clusters/pi-cluster/apps/

# Check Flux reconciliation status
flux get kustomizations -n flux-system
flux get sources -n flux-system

# Sync changes to cluster
flux reconcile kustomization apps -n flux-system

# Check cluster status
kubectl get nodes
kubectl get pods --all-namespaces

# Validate CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# Check certificate status
kubectl get certificates -A
kubectl get certificaterequests -A
```

## Cluster Management

### Multi-Node Cluster Operations
```bash
# Drain node for maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node after maintenance
kubectl uncordon <node-name>

# Check node status and resources
kubectl top nodes
kubectl describe nodes

# Monitor cluster health
kubectl get componentstatuses
kubectl get cs  # deprecated but still useful
```

### High Availability Setup
- Dual-node cluster: 192.168.254.22 (primary), 192.168.254.84 (secondary)
- Use NFS for shared persistent storage
- Configure CoreDNS with both node IPs for service resolution
- Implement load balancing for critical services

### Node Maintenance Procedures
1. Schedule maintenance windows during low usage
2. Drain node gracefully before updates
3. Verify all pods reschedule successfully
4. Monitor for any stuck pods or PVC issues
5. Test application functionality after maintenance

## Backup & Disaster Recovery

### etcd Backup Procedures
```bash
# Create etcd snapshot (k3s specific)
sudo k3s etcd-snapshot save --name "backup-$(date +%Y%m%d-%H%M%S)"

# List snapshots
sudo k3s etcd-snapshot ls

# Restore from snapshot
sudo k3s server --cluster-reset --restore-from=/var/lib/rancher/k3s/server/db/snapshots/snapshot-xxxx
```

### NFS Storage Backup
```bash
# Backup NFS shares
rsync -av --delete /path/to/nfs/share/ /backup/location/nfs-$(date +%Y%m%d)/

# Verify backup integrity
find /backup/location/nfs-$(date +%Y%m%d)/ -type f -exec md5sum {} \; > /backup/checksums-$(date +%Y%m%d).txt
```

### Application Data Backup
- Use Kubernetes CronJobs for automated backups
- Backup databases before major updates
- Test restore procedures regularly
- Store backups off-cluster when possible

## Network Management

### CoreDNS Configuration
- Custom host entries managed in `coredns-config.yaml`
- DNS entries for both node IPs (192.168.254.22/84)
- Reload CoreDNS after configuration changes:
```bash
kubectl rollout restart deployment coredns -n kube-system
```

### Ingress Patterns
- Use Traefik as ingress controller
- Follow naming: `<app>.k3s.tophermayor.com`
- Implement HTTPS with Cloudflare certificates
- Use middleware for common functionality (auth, redirects)

### Network Troubleshooting
```bash
# Test DNS resolution
nslookup homepage.k3s.tophermayor.com

# Check network policies
kubectl get networkpolicies -A

# Debug connectivity
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
```

## Secrets Management

### External Secrets Strategy
```bash
# Install External Secrets Operator
kubectl apply -f https://github.com/external-secrets/external-secrets/releases/latest/download/external-secrets.yaml

# Create SecretStore for Cloudflare
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: cloudflare-store
spec:
  provider:
    cloudflare:
      apiToken:
        secretRef:
          name: cloudflare-api-token
          key: api-token
```

### Sealed Secrets (Alternative)
```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Seal a secret
kubeseal --format yaml --cert=pub-cert.pem < secret.yaml > sealed-secret.yaml
```

### Secret Rotation
- Implement automated rotation for API keys
- Use short-lived tokens when possible
- Monitor secret expiration dates
- Test secret renewal procedures

## Operational Procedures

### Rollback Strategies
```bash
# Flux rollback to previous commit
flux reconcile kustomization apps --with-source=<previous-commit-hash>

# Manual rollback with kubectl
kubectl rollout undo deployment/<app-name>

# Check rollback status
kubectl rollout status deployment/<app-name>
```

### Blue-Green Deployments
- Use separate namespaces for blue/green environments
- Implement service switching for traffic routing
- Monitor health during transition
- Keep previous version running until verification complete

### Canary Deployments
- Start with 5% traffic to new version
- Gradually increase based on health metrics
- Monitor error rates and response times
- Auto-rollback on failure detection

## Monitoring & Observability

### Log Aggregation
```bash
# View pod logs
kubectl logs -f <pod-name> -n <namespace>

# View all logs in namespace
kubectl logs -l app=<app-name> -n <namespace>

# Check system logs
sudo journalctl -u k3s
```

### Metrics Collection
- Use Prometheus for metrics collection
- Implement Grafana dashboards for visualization
- Monitor node resources (CPU, memory, disk)
- Track application-specific metrics

### Health Monitoring
```bash
# Check pod health
kubectl get pods -A --field-selector=status.phase!=Running

# Check node health
kubectl get nodes -o wide

# Monitor PVC status
kubectl get pvc -A
```

## Performance & Resource Management

### Resource Allocation
```yaml
# Example resource requests/limits
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Node Resource Monitoring
```bash
# Monitor node usage
kubectl top nodes
kubectl top pods -A

# Check resource quotas
kubectl describe quota
```

### Storage Performance
- Monitor NFS mount performance
- Use appropriate storage classes based on I/O requirements
- Implement storage monitoring alerts
- Regular storage cleanup procedures

## Security Hardening

### Network Policies
```yaml
# Example NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-netpol
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Pod Security Standards
- Use restricted PodSecurity standards where possible
- Implement runtime security monitoring
- Regular image vulnerability scanning
- Use non-root containers when available

### RBAC Best Practices
- Principle of least privilege
- Regular RBAC audits
- Use service accounts for applications
- Implement role binding reviews

## Troubleshooting Guide

### Common Issues & Solutions
```bash
# Pod stuck in Pending state
kubectl describe pod <pod-name>  # Check events and resource constraints

# PVC stuck in Pending state
kubectl get pv  # Check available storage
kubectl describe pvc <pvc-name>  # Check storage class and capacity

# Certificate issues
kubectl describe certificate <cert-name>
kubectl get certificaterequests -A
```

### Debugging Workflows
1. Check pod status and events
2. Review application logs
3. Verify resource availability
4. Test network connectivity
5. Validate configuration
6. Check security policies

### Recovery Procedures
- Maintain disaster recovery documentation
- Test recovery procedures quarterly
- Keep critical contact information accessible
- Document known issues and solutions

## Maintenance & Upgrades

### k3s Upgrade Procedures
```bash
# Check current version
k3s --version

# Upgrade k3s (Ubuntu/Debian)
curl -sfL https://get.k3s.io | sh -s - --upgrade

# Verify upgrade
kubectl get nodes
```

### Flux CD Upgrades
```bash
# Check Flux version
flux --version

# Upgrade Flux
flux install --export > flux-system.yaml
kubectl apply -f flux-system.yaml
```

### Application Updates
- Test updates in staging environment first
- Use semantic versioning for releases
- Implement automated testing where possible
- Document breaking changes

## Code Style Guidelines

### YAML/Kubernetes Manifests
- Use 2 spaces for indentation
- Follow Kubernetes API naming conventions (lowercase, hyphenated)
- Group related resources with `---` separator
- Include metadata.name and metadata.labels consistently
- Use app labels for pod selectors: `app: <app-name>`

### Resource Organization
- Separate PVC definitions from main app manifests
- Use ConfigMaps for configuration data
- Include Services and Ingress with Deployments
- Namespace: default (unless specified otherwise)
- Group apps by function in kustomization.yaml order
- Use NFS storage classes for persistent data on k3s

### Flux CD Specific
- Use Kustomize for resource management
- Reference resources in kustomization.yaml in logical order
- Include storage classes before PVCs before applications
- Flux automatically detects changes and applies to k3s cluster
- All changes should be committed to Git for cluster state management

### Naming Conventions
- Apps: lowercase, hyphenated (e.g., adguard, homepage)
- PVCs: `<app-name>-data` pattern
- ConfigMaps: `<app-name>-settings` pattern
- Services: match app name
- Ingress: match app name for routing
- Storage classes: descriptive names (nfs-storageclass, nfs-nextcloud-storageclass)

### Security
- Use specific image tags instead of latest when possible
- Implement resource requests/limits for production
- Use NetworkPolicies for sensitive applications
- Store secrets in Kubernetes Secret resources, not in Git
- Use cert-manager for TLS certificate management

## Environment Management

### Multi-Environment Strategy
```bash
# Environment-specific kustomization overlays
clusters/
├── pi-cluster/
│   ├── base/
│   ├── staging/
│   └── production/
```

### Configuration Management
- Use Kustomize overlays for environment differences
- Separate sensitive configuration from application config
- Implement configuration validation
- Use ConfigMaps for non-sensitive configuration

### Environment-Specific Overrides
```yaml
# Example kustomization.yaml for staging
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
patchesStrategicMerge:
- staging-deployment.yaml
```

## Homelab-Specific Considerations

### Power Management
- Implement UPS monitoring and alerts
- Configure graceful shutdown procedures
- Monitor power consumption
- Plan for power outage scenarios

### Hardware Monitoring
```bash
# Monitor CPU temperature
vcgencmd measure_temp

# Check disk health
sudo smartctl -a /dev/sda

# Monitor memory usage
free -h
```

### Storage Health
- Regular disk health checks
- Monitor NFS server status
- Implement storage capacity alerts
- Plan storage expansion procedures

### Network Reliability
- Monitor network connectivity between nodes
- Implement network redundancy where possible
- Monitor bandwidth usage
- Document network topology

## Emergency Procedures

### Cluster Recovery
1. Assess the scope of the failure
2. Check node status and connectivity
3. Review recent changes in Git
4. Restore from latest backup if needed
5. Verify application functionality
6. Document the incident and resolution

### Communication Plan
- Maintain contact information for all stakeholders
- Document escalation procedures
- Create incident response templates
- Schedule post-incident reviews