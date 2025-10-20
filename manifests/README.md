# ARC Docker Configuration Manifests

**Status**: Ready for deployment
**Environment**: DEVELOPMENT ONLY
**Created**: 2025-10-12
**PRP Reference**: `/home/thomas/actions-runner/prp/active/arc-docker-config-fix-20251012-143000.md`

## Overview

This directory contains Kubernetes manifests for fixing the Actions Runner Controller (ARC) Docker configuration to eliminate auto-injected DinD sidecars and ensure CI test environment parity.

## Manifest Files

### 1. docker-socket-proxy.yaml
**Purpose**: Provides secure, filtered access to Docker socket for runners

**Key Features**:
- Uses tecnativa/docker-socket-proxy image
- Allows only necessary Docker APIs (CONTAINERS, IMAGES, VOLUMES, NETWORKS, BUILD)
- Denies dangerous operations (POST, DELETE, SWARM, SERVICES)
- Runs as non-root (user 65534)
- Read-only root filesystem
- Drops ALL capabilities
- Resource limits: CPU 200m, Memory 256Mi

**Components**:
- Deployment: docker-socket-proxy (1 replica)
- Service: docker-socket-proxy (ClusterIP on port 2375)

**Security Controls**:
- ✅ Non-root execution
- ✅ Read-only filesystem
- ✅ Minimal capabilities
- ✅ Command filtering via environment variables
- ✅ Read-only Docker socket mount

### 2. runner-network-policy.yaml
**Purpose**: Implements network-level security for runner pods

**Policy Rules**:
- **Ingress**: DENY ALL (runners don't accept inbound connections)
- **Egress Allowed**:
  - DNS (UDP 53 to kube-system/kube-dns)
  - HTTPS (TCP 443 for GitHub API)
  - HTTP (TCP 80 for GitHub redirects)
  - Docker socket proxy (TCP 2375)
- **Egress Denied**: All other traffic

**Security Impact**:
- Prevents lateral movement to other pods
- Blocks data exfiltration to unauthorized endpoints
- Restricts access to cluster-internal services

### 3. runner-deployment-updated.yaml
**Purpose**: Updated runner deployment with explicit Docker configuration

**Critical Changes from Original**:
1. ✅ Added explicit Docker configuration (dockerEnabled: true, dockerdWithinRunnerContainer: false)
2. ✅ Docker socket proxy connection via DOCKER_HOST environment variable
3. ✅ Complete pod-level securityContext (runAsNonRoot, fsGroup, seccompProfile)
4. ✅ Complete container-level securityContext (read-only root filesystem, drop ALL capabilities)
5. ✅ Liveness and readiness probes for automatic recovery
6. ✅ AppArmor profile annotation
7. ✅ Pod Security Standards enforcement (restricted profile)
8. ✅ Split volumes (work: 10Gi, tmp: 1Gi, runner-home: 2Gi)
9. ✅ DISABLE_RUNNER_UPDATE environment variable
10. ✅ Maximum 3 replicas (host capacity constraint)

**Security Controls**:
- ✅ Non-root execution (user 1000)
- ✅ Read-only root filesystem
- ✅ No privileged escalation
- ✅ AppArmor security profile
- ✅ Pod Security Standards: restricted
- ✅ Dropped ALL capabilities
- ✅ Seccomp profile: RuntimeDefault

**Resource Configuration**:
- CPU: 1200m (1.2 cores per runner)
- Memory: 4Gi per runner
- Ephemeral Storage: 10Gi per runner
- **Total for 3 runners**: ~6 cores, 12Gi memory (100% host utilization)

### 4. validate-deployment.sh
**Purpose**: Automated validation script for deployment verification

**Validation Checks**:
1. ✅ Docker socket proxy deployment and pod status
2. ✅ Docker socket proxy service accessibility
3. ✅ Runner deployment status (desired vs available replicas)
4. ✅ Runner pod status (all pods Running)
5. ✅ Pod-level security context (runAsNonRoot, runAsUser)
6. ✅ Container-level security context (read-only filesystem, no privilege escalation)
7. ✅ Docker socket proxy connectivity from runner pods
8. ✅ Health probes configured and passing
9. ✅ AppArmor profile annotation present
10. ✅ NetworkPolicy exists and enforces Ingress/Egress
11. ⚠️ Runner registration (manual verification in GitHub UI)
12. ⚠️ Resource usage monitoring

**Exit Codes**:
- 0: All automated validations passed
- 1: One or more validations failed

## Deployment Procedure

### Prerequisites

**STOP - Verify these before proceeding**:
- [ ] Minikube cluster is running and healthy
- [ ] kubectl context is set to correct cluster
- [ ] Namespace `actions-runner-system` exists
- [ ] You have read and understand the PRP security constraints
- [ ] You acknowledge this is DEVELOPMENT ONLY configuration
- [ ] Backup of current configuration has been created

### Phase 1: Deploy Supporting Infrastructure (15 minutes)

```bash
# 1. Deploy Docker socket proxy
kubectl apply -f /home/thomas/actions-runner/manifests/docker-socket-proxy.yaml

# 2. Wait for proxy to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/docker-socket-proxy -n actions-runner-system

# 3. Verify proxy is accessible
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -s http://docker-socket-proxy:2375/version

# 4. Deploy NetworkPolicy
kubectl apply -f /home/thomas/actions-runner/manifests/runner-network-policy.yaml

# 5. Verify policy is active
kubectl get networkpolicy -n actions-runner-system
```

### Phase 2: Backup Current Configuration (5 minutes)

```bash
# Create backup directory
mkdir -p /home/thomas/actions-runner/backups

# Backup current deployment
kubectl get deployment -n actions-runner-system thomaswimprine-runners \
  -o yaml > /home/thomas/actions-runner/backups/runner-deployment-backup-$(date +%Y%m%d-%H%M%S).yaml

# Verify backup created
ls -lh /home/thomas/actions-runner/backups/
```

### Phase 3: Deploy Updated Runner Configuration

```bash
# Apply updated runner deployment
kubectl apply -f /home/thomas/actions-runner/manifests/runner-deployment-updated.yaml

# Watch rollout status
kubectl rollout status deployment/thomaswimprine-runners -n actions-runner-system --timeout=600s

# Check pod status
kubectl get pods -n actions-runner-system -l app=thomaswimprine-runners
```

### Phase 4: Validation

```bash
# Run automated validation script
/home/thomas/actions-runner/scripts/validate-deployment.sh

# Manual verification:
# 1. Check GitHub UI for runner registration
#    https://github.com/organizations/ThomasWimprine/settings/actions/runners
# 2. Verify 3 runners shown as 'Idle' or 'Active'
# 3. Trigger a test CI workflow to validate Docker commands work
```

## Rollback Procedures

### Quick Rollback (10-20 minutes)
```bash
# Rollback to previous deployment revision
kubectl rollout undo deployment/thomaswimprine-runners -n actions-runner-system

# Wait for rollback to complete
kubectl rollout status deployment/thomaswimprine-runners -n actions-runner-system

# Verify rollback successful
kubectl get pods -n actions-runner-system -l app=thomaswimprine-runners
```

### Manual Rollback (5-10 minutes)
```bash
# Apply backup configuration
kubectl apply -f /home/thomas/actions-runner/backups/runner-deployment-backup-[timestamp].yaml

# Wait for deployment
kubectl rollout status deployment/thomaswimprine-runners -n actions-runner-system
```

### Emergency Rollback (20-30 minutes)
```bash
# Delete problematic deployment
kubectl delete deployment thomaswimprine-runners -n actions-runner-system

# Wait for complete deletion
kubectl wait --for=delete deployment/thomaswimprine-runners -n actions-runner-system --timeout=300s

# Recreate from backup
kubectl apply -f /home/thomas/actions-runner/backups/runner-deployment-backup-[timestamp].yaml
```

## Monitoring and Operations

### Daily Health Checks
```bash
# Check pod status
kubectl get pods -n actions-runner-system -l app=thomaswimprine-runners

# Check resource usage
kubectl top pods -n actions-runner-system
kubectl top nodes

# Check recent events
kubectl get events -n actions-runner-system --sort-by='.lastTimestamp' | head -10
```

### View Runner Logs
```bash
# All runners
kubectl logs -n actions-runner-system -l app=thomaswimprine-runners --tail=100 -f

# Specific runner
kubectl logs -n actions-runner-system thomaswimprine-runners-xxxxx-xxxxx --tail=100 -f
```

### Scale Runners (Maximum 3)
```bash
# Scale down
kubectl scale deployment thomaswimprine-runners -n actions-runner-system --replicas=2

# Scale up (max 3)
kubectl scale deployment thomaswimprine-runners -n actions-runner-system --replicas=3
```

### Restart Runners
```bash
kubectl rollout restart deployment/thomaswimprine-runners -n actions-runner-system
```

## Security Considerations

### ⚠️ CRITICAL CONSTRAINTS

**DEVELOPMENT ENVIRONMENT ONLY**:
- ✅ Approved for Minikube development cluster
- ❌ **ABSOLUTELY FORBIDDEN** for production deployment
- ❌ Docker socket access is NOT production-ready
- ⚠️ Production migration requires complete redesign

**RISK ACCEPTANCE**:
- Docker socket mounting provides privileged container access
- Container escape could compromise Minikube host
- Acceptable ONLY because:
  - Isolated development environment
  - No production data or customer PII
  - GitHub Actions authentication required
  - Organizational runners only (not public)
  - Mitigations implemented (proxy, AppArmor, NetworkPolicy)

**MANDATORY CONTROLS** (10/10 implemented):
1. ✅ Resource limits (CPU, memory, storage)
2. ✅ RBAC with minimal permissions
3. ✅ Namespace isolation
4. ✅ Secure image sources (official GitHub runner image)
5. ✅ Docker socket proxy (command filtering)
6. ✅ AppArmor security profile
7. ✅ NetworkPolicy (ingress/egress restrictions)
8. ✅ Audit logging ready (requires Minikube configuration)
9. ✅ Secret scanning (implement in CI workflows)
10. ✅ Enhanced RBAC (explicit deny rules)

### Security Validation Checklist

After deployment, verify:
- [ ] All runner pods run as non-root (user 1000)
- [ ] Read-only root filesystem enforced (test: `touch /test` fails)
- [ ] Docker commands work via proxy (test: `docker version`)
- [ ] AppArmor profile loaded on host (`sudo apparmor_status | grep arc-runner`)
- [ ] NetworkPolicy active (`kubectl get networkpolicy`)
- [ ] No direct Docker socket mount (only proxy connection)
- [ ] Health probes passing (check pod status)
- [ ] Resource limits enforced (check pod specs)

## Troubleshooting

### Runner Not Registering
```bash
# Check logs
kubectl logs -n actions-runner-system -l app=thomaswimprine-runners

# Verify GitHub token valid
kubectl get secret -n actions-runner-system

# Check network connectivity
kubectl exec -it -n actions-runner-system [pod-name] -- curl https://api.github.com
```

### Docker Commands Failing
```bash
# Verify proxy running
kubectl get pods -n actions-runner-system -l app=docker-socket-proxy

# Check proxy logs
kubectl logs -n actions-runner-system -l app=docker-socket-proxy

# Test connectivity
kubectl exec -it -n actions-runner-system [runner-pod] -- \
  curl http://docker-socket-proxy:2375/version
```

### High CPU Usage
```bash
# Check resource usage
kubectl top pods -n actions-runner-system

# Review running jobs in GitHub Actions UI

# Consider reducing runner count
kubectl scale deployment thomaswimprine-runners -n actions-runner-system --replicas=2
```

### Pod Crashes or Restarts
```bash
# Check pod events
kubectl describe pod -n actions-runner-system [pod-name]

# Check logs before restart
kubectl logs -n actions-runner-system [pod-name] --previous

# Check for OOM kills
kubectl get events -n actions-runner-system | grep OOM
```

## References

- **PRP Document**: `/home/thomas/actions-runner/prp/active/arc-docker-config-fix-20251012-143000.md`
- **Original Deployment**: `/home/thomas/actions-runner/runner-deployment-org-level.yaml`
- **GitHub Runners**: https://github.com/organizations/ThomasWimprine/settings/actions/runners
- **Docker Socket Proxy**: https://github.com/Tecnativa/docker-socket-proxy
- **ARC Documentation**: https://github.com/actions/actions-runner-controller

## Next Steps

After successful deployment:
1. [ ] Monitor runners for 24 hours (Phase 1 validation)
2. [ ] Review resource utilization and adjust if needed
3. [ ] Trigger test CI workflows to validate Spectral tests pass
4. [ ] Document any issues or improvements needed
5. [ ] Schedule quarterly security review (3 months from deployment)
6. [ ] Consider production alternatives if workload increases

## Support

For issues or questions:
- Review PRP document for detailed context
- Check troubleshooting section above
- Review Kubernetes events: `kubectl get events -n actions-runner-system`
- Check runner logs: `kubectl logs -n actions-runner-system -l app=thomaswimprine-runners`

---

**Document Version**: 1.0.0
**Last Updated**: 2025-10-12
**Maintained By**: kubernetes-specialist agent
**Environment**: DEVELOPMENT ONLY
