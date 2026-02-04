# Claude Code Instructions

## CRITICAL SAFETY RULES

### 1. Verify Tunnelblick is DISCONNECTED
Before running ANY kubectl command, verify Tunnelblick has no active VPN connection:
```bash
# Check if Tunnelblick VPN is connected
pgrep -f "openvpn" && echo "WARNING: VPN ACTIVE - DO NOT PROCEED" || echo "OK: No VPN active"
```

This prevents accidentally running commands against the wrong cluster.

### 2. ALWAYS use explicit kubeconfig
NEVER run kubectl without the `--kubeconfig` flag:
```bash
# CORRECT
kubectl --kubeconfig .kube/config get pods

# WRONG - could hit wrong cluster
kubectl get pods
```

## Server Access

### Get Tailscale IP for Hetzner Cloud server
```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale status
```

### SSH into k3s server
```bash
ssh -i .ssh/rdocchio root@<TAILSCALE_IP>
```

Example:
```bash
ssh -i .ssh/rdocchio root@100.94.96.73
```

### Run remote commands
```bash
ssh -i .ssh/rdocchio root@100.94.96.73 "<command>"
```

## Kubernetes Access

### kubectl commands
```bash
kubectl --kubeconfig .kube/config <command>
```

Example:
```bash
kubectl --kubeconfig .kube/config get nodes
kubectl --kubeconfig .kube/config get pods -A
```

### Kubeconfig uses MagicDNS
The kubeconfig connects to `k3s-dev-01-1.taild34517.ts.net:6443` via Tailscale MagicDNS.

## Git Access

### Remote repository
```
origin  git@github.com:Startup-AI-Infrastructure/apps-helm-chart-template.git
```

### SSH key configuration
Git is configured to use the local SSH key for GitHub:
```bash
git config core.sshCommand "ssh -i .ssh/rdocchio -o IdentitiesOnly=yes"
```

This allows `git push` and `git pull` to work without additional parameters.

## Branch Strategy

This repository uses feature branches for development:

- Create feature branches for new features/changes
- Push branch and create PR to `main`
- Chart releases are triggered by git tags (e.g., `v1.0.0`)

## Helm Chart Development

### Lint the chart
```bash
helm lint charts/app
```

### Test a preset
```bash
helm template test charts/app --set preset=api --set image.repository=myapp
```

### Available presets
- `api` - Deployment + Service + Ingress + HPA
- `worker` - Deployment + HPA
- `job` - Job for one-time tasks
- `cronjob` - CronJob for scheduled tasks
- `daemon` - DaemonSet for node-level workloads

### Release process
1. Update `charts/app/Chart.yaml` version if needed
2. Create and push a git tag: `git tag v1.0.0 && git push origin v1.0.0`
3. GitHub Actions will publish to `oci://ghcr.io/startup-ai-infrastructure/charts/app`
