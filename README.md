# Actions Runner Controller (ARC) - Self-Hosted Runners

**Organization**: ThomasWimprine
**Environment**: Development (Minikube)
**Deployment Date**: [DEPLOYMENT_DATE_PLACEHOLDER]
**Status**: Operational

---

## Overview

This repository contains the configuration and operational documentation for self-hosted GitHub Actions runners deployed on Minikube using Actions Runner Controller (ARC). The runners provide consistent CI/CD execution environment for the ThomasWimprine organization.

**Key Features**:
- 3 organization-level runners (maximum capacity for 6-core host)
- Docker socket access via secure proxy (development environment only)
- Comprehensive security controls (AppArmor, NetworkPolicy, RBAC)
- TDD pipeline integration with 6 validation gates
- Spectral CLI for API contract validation

---

## Quick Start

```bash
# Check runner status
kubectl get pods -n actions-runner-system -l app=thomaswimprine-runners

# View runner logs
kubectl logs -n actions-runner-system -l app=thomaswimprine-runners --tail=100

# Scale runners (max 3)
kubectl scale deployment thomaswimprine-runners -n actions-runner-system --replicas=3

# Run health check
/home/thomas/actions-runner/scripts/daily-health-check.sh
```

---

## Docker Configuration

**Status**: Deployed on [DEPLOYMENT_DATE_PLACEHOLDER]
**Configuration**: Docker socket proxy (development environment only)
**Runners**: 3 replicas (maximum for 6-core host)
**Security**: AppArmor profile, NetworkPolicy, RBAC restrictions

### Architecture

The runner deployment uses a **Docker socket proxy** architecture to provide controlled Docker access:

```
┌─────────────────────────────────────────────────────┐
│                    GitHub Actions                    │
│             (Job Queue & Orchestration)              │
└───────────────────────┬─────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────┐
│         Kubernetes Cluster (Minikube)                │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Runner 1   │  │   Runner 2   │  │ Runner 3  │ │
│  │              │  │              │  │           │ │
│  │  Docker CLI  │  │  Docker CLI  │  │Docker CLI │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                  │                 │       │
│         └──────────────────┼─────────────────┘       │
│                            ↓                          │
│                 ┌─────────────────────┐              │
│                 │ Docker Socket Proxy │              │
│                 │  (Command Filter)   │              │
│                 └──────────┬──────────┘              │
│                            ↓                          │
│                    /var/run/docker.sock              │
│                    (Minikube Host)                    │
└─────────────────────────────────────────────────────┘
```

**Why Docker Socket Proxy?**
- **Security**: Filters dangerous Docker commands (prevent privileged escalation)
- **Auditability**: Logs all Docker API requests
- **Control**: Allows only necessary Docker operations (build, run, stop)
- **Isolation**: Prevents direct host socket access from runner containers

### Critical Constraints

⚠️ **READ THIS BEFORE MAKING ANY CHANGES**:

1. **DEVELOPMENT ENVIRONMENT ONLY**
   - ✅ Approved for Minikube development cluster
   - ❌ **ABSOLUTELY FORBIDDEN** for production deployment
   - ❌ Docker socket mounting is NOT production-ready
   - ⚠️ Production migration requires complete redesign

2. **RESOURCE LIMITATIONS**
   - Maximum 3 concurrent runners (NOT more)
   - 6-core host cannot support > 3 runners
   - Each runner requires ~2 cores (1.2 CPU request + overhead)
   - **DO NOT scale beyond 3 replicas** - system will become unstable

3. **SECURITY REQUIREMENTS**
   - All 10 security controls MUST remain active (see Security section)
   - Docker socket proxy REQUIRED - never use direct socket mounting
   - AppArmor profile MUST be enforced
   - Network policies MUST be active
   - RBAC restrictions MUST not be weakened

4. **PRODUCTION MIGRATION PATH**
   - When production deployment needed, use alternatives:
     - Docker-in-Docker (DinD) sidecar with proper isolation
     - Kaniko for rootless container builds
     - Rootless DinD with Podman
     - BuildKit with rootless mode
   - **DO NOT migrate current configuration to production**

### Deployment Details

**Deployed Components**:
- 3× Runner pods (`thomaswimprine-runners-*`)
- 1× Docker socket proxy pod (`docker-socket-proxy-*`)
- Network policies for ingress/egress control
- AppArmor profile (`arc-runner`) on Minikube host
- RBAC roles and bindings for least-privilege access

**Resource Allocation**:
- **Per Runner**: 1.2 CPU cores, 4GB RAM, 10Gi ephemeral storage
- **Total Cluster**: ~6 cores, ~12GB RAM (3 runners + overhead)
- **Host Capacity**: 6 cores, 16GB RAM (Minikube single-node)

**Security Controls** (10/10 active):
1. ✅ Resource limits (CPU, memory, storage)
2. ✅ RBAC configuration (ServiceAccount, Role, RoleBinding)
3. ✅ Namespace isolation (`actions-runner-system`)
4. ✅ Secure image source (official GitHub runner images)
5. ✅ Docker socket proxy (tecnativa/docker-socket-proxy)
6. ✅ AppArmor profile (restricts dangerous syscalls)
7. ✅ Network policies (ingress/egress restrictions)
8. ✅ Audit logging (Kubernetes audit logs)
9. ✅ Secret scanning (Spectral/TruffleHog in workflows)
10. ✅ Enhanced RBAC (explicit deny rules)

### Configuration Files

**Primary Configuration**:
- `runner-deployment-org-level.yaml` - Main runner deployment manifest
- `security/docker-socket-proxy.yaml` - Docker socket proxy deployment
- `security/runner-network-policy.yaml` - Network policy restrictions
- `security/apparmor-profile.txt` - AppArmor security profile

**Operational Documentation**:
- `docs/RUNBOOK.md` - Complete operational procedures and troubleshooting
- `docs/DEPLOYMENT_LOG.md` - Deployment tracking template
- `prp/active/arc-docker-config-fix-20251012-143000.md` - Product Requirements Proposal

### Common Operations

**View Runner Status**:
```bash
kubectl get pods -n actions-runner-system -l app=thomaswimprine-runners -o wide
```

**Check GitHub Registration**:
```bash
# Via CLI
gh api /orgs/ThomasWimprine/actions/runners

# Via Web UI
https://github.com/organizations/ThomasWimprine/settings/actions/runners
```

**Scale Runners** (max 3):
```bash
kubectl scale deployment thomaswimprine-runners -n actions-runner-system --replicas=2
```

**Restart Runners**:
```bash
kubectl rollout restart deployment/thomaswimprine-runners -n actions-runner-system
```

**View Logs**:
```bash
kubectl logs -n actions-runner-system -l app=thomaswimprine-runners --tail=100 -f
```

**Check Docker Socket Proxy**:
```bash
kubectl get pods -n actions-runner-system -l app=docker-socket-proxy
kubectl logs -n actions-runner-system -l app=docker-socket-proxy
```

**Run Health Check**:
```bash
/home/thomas/actions-runner/scripts/daily-health-check.sh
```

**Run Security Audit**:
```bash
/home/thomas/actions-runner/scripts/weekly-security-audit.sh
```

### Troubleshooting

For detailed troubleshooting procedures, see: **[docs/RUNBOOK.md](docs/RUNBOOK.md)**

**Common Issues**:
- **Runner not registering**: Check GitHub token, network connectivity, logs
- **Docker commands failing**: Check proxy status, connectivity, network policies
- **High CPU usage**: Scale down runners, check for runaway jobs
- **Jobs queuing**: Scale up runners (max 3), check labels

**Emergency Contacts**:
- DevOps Engineer: [Contact details]
- Security Team: [Contact details]
- Slack: #devops-alerts, #incidents

---

## Repository Structure

```
/home/thomas/actions-runner/
├── runner-deployment-org-level.yaml    # Main runner deployment
├── docs/
│   ├── RUNBOOK.md                      # Operational procedures (~200 lines)
│   └── DEPLOYMENT_LOG.md               # Deployment tracking template
├── security/
│   ├── docker-socket-proxy.yaml        # Docker socket proxy deployment
│   ├── runner-network-policy.yaml      # Network restrictions
│   └── apparmor-profile.txt            # AppArmor security profile
├── scripts/
│   ├── daily-health-check.sh           # Daily health monitoring
│   ├── weekly-security-audit.sh        # Weekly security validation
│   └── verify-security-controls.sh     # Security controls verification
├── prp/
│   └── active/
│       └── arc-docker-config-fix-*.md  # Product Requirements Proposal
├── backups/                             # Configuration backups
└── manifests/                           # Additional Kubernetes manifests
```

---

## TDD Pipeline Integration

The runners are integrated with a comprehensive TDD enforcement pipeline with 6 validation gates:

**Gate 1: TDD Cycle Verification**
- Enforces RED-GREEN-REFACTOR pattern
- Validates commit history for proper TDD sequence

**Gate 2: Mock Detection (Production Isolation)**
- Zero mocks allowed in `src/` directories
- Ensures production code is not contaminated with test doubles

**Gate 3: API Contract Validation (Spectral)**
- 100% API endpoint contract coverage
- OpenAPI/GraphQL schema validation
- **Critical**: This gate was failing before Docker configuration fix

**Gate 4: Test Coverage**
- 100% coverage required (lines, branches, functions, statements)
- No untested code allowed

**Gate 5: Mutation Testing**
- Mutation score ≥95% required
- Validates test effectiveness

**Gate 6: Quality Gate Aggregation**
- All gates must pass simultaneously
- Aggregated quality check

**TDD Pipeline Compatibility**: ✅ FULL
- No workflow modifications needed
- Spectral validation works correctly with Docker socket proxy
- All 6 gates function as expected

---

## Security

### Risk Rating

- **Development Environment**: HIGH (acceptable with controls)
- **Production Environment**: CRITICAL (FORBIDDEN)

### Threat Model

| Threat | Likelihood | Impact | Severity | Mitigation |
|--------|-----------|--------|----------|------------|
| Container Escape | Medium | Critical | HIGH | AppArmor + Proxy |
| Privilege Escalation | Medium | Critical | HIGH | RBAC + Security Context |
| Data Exfiltration | Low | High | MEDIUM | Network Policies |
| Denial of Service | Medium | Medium | MEDIUM | Resource Limits |
| Lateral Movement | Low | High | MEDIUM | Network Policies + RBAC |
| Audit Bypass | High | Medium | MEDIUM | Audit Logging |
| Supply Chain | Low | Critical | MEDIUM | Image Scanning |

### Security Acceptance

**Accepted for Development**:
- Docker socket mounting risk accepted due to:
  - ✅ Isolated development environment
  - ✅ No production data or customer PII
  - ✅ GitHub Actions authentication required
  - ✅ Organizational runners only (not public)
  - ✅ All 10 security controls implemented

**NOT Accepted for Production**:
- ❌ Docker socket mounting FORBIDDEN in production
- ❌ Privilege escalation risk unacceptable
- ❌ Container escape blast radius too high
- ❌ Compliance requirements cannot be met

### Quarterly Security Review

**Next Review**: [DEPLOYMENT_DATE + 3 months]

**Review Agenda**:
- Evaluate production readiness
- Review security incidents (if any)
- Update threat model
- Assess alternative solutions (DinD, Kaniko)
- Verify all 10 controls still active

---

## Monitoring & Alerting

### Key Metrics

| Metric | Warning Threshold | Critical Threshold |
|--------|-------------------|-------------------|
| Runner Pod Status | < 3 Running | < 2 Running |
| Host CPU Usage | > 80% | > 90% |
| Host Memory Usage | > 10GB | > 12GB |
| Runner CPU Usage | > 1.2 cores | > 1.5 cores |
| Runner Memory Usage | > 4GB | > 4.5GB |
| Docker Proxy Errors | > 5/min | > 20/min |
| Network Policy Drops | > 10/min | > 100/min |
| AppArmor Denials | > 5/min | > 20/min |
| Job Success Rate | < 95% | < 90% |

### Monitoring Schedule

**Daily** (9:00 AM):
- Run health check script
- Verify all runners online
- Check resource usage
- Review recent events

**Weekly** (Monday 10:00 AM):
- Run security audit script
- Verify all security controls active
- Review AppArmor denials
- Check network policy drops

**Monthly** (1st of month):
- Capacity review
- Job queue time analysis
- Peak resource usage review
- Runner utilization assessment

### Alerting

**P0 - Critical** (Immediate Response):
- All runners down
- Security breach detected
- Cluster unresponsive

**P1 - High** (30 min Response):
- Majority runners down
- All jobs failing (>90% failure rate)
- Docker socket proxy down

**P2 - Medium** (2 hour Response):
- Single runner down
- High resource usage (80-90%)
- Jobs queuing but processing

**P3 - Low** (8 hour Response):
- Minor configuration drift
- Single job failure
- Non-impacting warnings

---

## Rollback Procedures

### Automated Rollback (10-20 minutes)

```bash
kubectl rollout undo deployment/thomaswimprine-runners -n actions-runner-system
kubectl rollout status deployment/thomaswimprine-runners -n actions-runner-system
```

### Manual Rollback (5-10 minutes)

```bash
kubectl apply -f /home/thomas/actions-runner/backups/runner-deployment-backup-[timestamp].yaml
kubectl rollout status deployment/thomaswimprine-runners -n actions-runner-system
```

### Emergency Rollback (20-30 minutes)

```bash
kubectl delete deployment thomaswimprine-runners -n actions-runner-system
kubectl wait --for=delete deployment/thomaswimprine-runners -n actions-runner-system --timeout=300s
kubectl apply -f /home/thomas/actions-runner/backups/runner-deployment-backup-[timestamp].yaml
```

### Nuclear Option (60-77 minutes)

**LAST RESORT - Complete cluster rebuild**

See: [docs/RUNBOOK.md - Emergency Procedure 4](docs/RUNBOOK.md#emergency-procedure-4-nuclear-option-cluster-rebuild)

---

## Maintenance

### Backup Schedule

**Before any change**:
```bash
kubectl get deployment thomaswimprine-runners -n actions-runner-system -o yaml \
  > /home/thomas/actions-runner/backups/runner-deployment-backup-$(date +%Y%m%d-%H%M%S).yaml
```

**Retention**: Keep backups for 90 days

### Update Procedure

**Runner Image Updates**:
1. Check for new runner versions: `gh api /repos/actions/runner/releases/latest`
2. Update image tag in `runner-deployment-org-level.yaml`
3. Test on single runner (scale to 1, update, validate 24 hours)
4. Scale to 3 runners after validation

**Security Control Updates**:
1. Create backup of current configuration
2. Test changes on non-production cluster (if available)
3. Apply changes during low-traffic period
4. Verify all 10 controls still active
5. Run security audit script

**Kubernetes Upgrades**:
1. Backup all configurations
2. Test upgrade on test cluster
3. Upgrade Minikube: `minikube delete && minikube start --cpus=6 --memory=16384`
4. Redeploy all components
5. Validate runners operational

---

## Documentation

**Operational**:
- [RUNBOOK.md](docs/RUNBOOK.md) - Complete operational procedures and troubleshooting
- [DEPLOYMENT_LOG.md](docs/DEPLOYMENT_LOG.md) - Deployment tracking template

**Planning**:
- [PRP](prp/active/arc-docker-config-fix-20251012-143000.md) - Product Requirements Proposal

**Security**:
- [Docker Socket Proxy Config](security/docker-socket-proxy.yaml)
- [Network Policy](security/runner-network-policy.yaml)
- [AppArmor Profile](security/apparmor-profile.txt)

**Historical**:
- Previous deployment logs: `deployments/`
- Configuration backups: `backups/`

---

## FAQ

**Q: Can I scale to more than 3 runners?**
A: No. The 6-core host cannot support more than 3 runners (each requires ~2 cores). Attempting to scale beyond 3 will cause system instability and pod evictions.

**Q: Can I use this configuration in production?**
A: Absolutely not. Docker socket mounting is forbidden in production due to security risks. See Production Migration Path in Docker Configuration section.

**Q: Why is Docker socket access needed?**
A: Spectral CLI and other CI tools require Docker to build/test containers. The socket proxy provides controlled, audited Docker access.

**Q: What happens if Docker socket proxy goes down?**
A: All jobs requiring Docker will fail. Runners will remain online but cannot execute Docker commands. Restore proxy immediately (see Runbook).

**Q: How do I check if all security controls are active?**
A: Run `/home/thomas/actions-runner/scripts/verify-security-controls.sh` to verify all 10 controls.

**Q: Can I disable network policies temporarily for troubleshooting?**
A: Only in emergency situations. Document the reason, restore immediately after troubleshooting, and run security audit.

**Q: Where are the audit logs?**
A: Kubernetes audit logs: `minikube ssh -- sudo tail /var/log/kubernetes/audit.log`
Docker proxy logs: `kubectl logs -n actions-runner-system -l app=docker-socket-proxy`

**Q: How do I handle a security incident?**
A: Execute Emergency Stop procedure (`kubectl scale --replicas=0`), notify security team immediately, preserve logs, follow incident response protocol.

---

## Support & Contact

**Primary Contacts**:
- **DevOps Engineer**: [Contact details]
- **Security Team**: [Contact details]
- **Platform Team**: [Contact details]

**Communication Channels**:
- **Slack - Incidents**: #incidents
- **Slack - DevOps Alerts**: #devops-alerts
- **Slack - Platform**: #platform-team
- **Email**: devops@example.com

**On-Call**:
- Check PagerDuty for current on-call engineer
- P0 incidents: Page immediately
- P1 incidents: Slack #devops-alerts

---

## License

[LICENSE TYPE] - Internal use only

---

## Changelog

### [DEPLOYMENT_DATE_PLACEHOLDER] - Docker Socket Proxy Configuration
- **Added**: Docker socket proxy for secure Docker access
- **Added**: AppArmor security profile enforcement
- **Added**: Network policies for ingress/egress control
- **Added**: Enhanced RBAC with explicit deny rules
- **Fixed**: Spectral CLI failures in CI (was caused by dind auto-injection)
- **Changed**: Maximum runners reduced to 3 (resource constraint)
- **Security**: All 10 mandatory security controls implemented

### [Previous Date] - Initial Deployment
- Initial runner deployment with 3 organization-level runners
- Basic RBAC and namespace isolation
- Resource limits configured

---

**Document Version**: 1.0.0
**Last Updated**: [DEPLOYMENT_DATE_PLACEHOLDER]
**Next Review**: [DEPLOYMENT_DATE_PLACEHOLDER + 3 months]
**Owner**: Platform Team

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
