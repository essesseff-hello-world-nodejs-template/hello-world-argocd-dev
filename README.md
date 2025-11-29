# hello-world - Argo CD Application (DEV)

This repository contains the Argo CD Application manifest for the **DEV** environment of the hello-world essesseff app.

## Repository Structure

```
hello-world-argocd-dev/
├── app-of-apps.yaml                  # Root Application (apply this to Argo CD)
├── argocd/
│   └── hello-world-dev-application.yaml  # DEV environment Application manifest (auto-synced)
├── argocd-repository-secret.yaml     # Argo CD repository secrets (configure before applying)
├── ghcr-credentials-secret.yaml      # GHCR credentials (set once for organization)
└── README.md                          # This file
```

## Architecture

- **Deployment Model**: Trunk-based development (single `main` branch)
- **Auto-Deploy**: Enabled (via essesseff webhooks)
- **GitOps**: Managed by Argo CD with automated sync

## Quick Start

### Deploy to Argo CD

1. **Configure Argo CD repository access** (if not already done):
   ```bash
   # Edit argocd-repository-secret.yaml with your GitHub Argo CD machine username and token
   # ***BE SURE TO DELETE THE FILE AFTERWARDS SO AS NOT TO COMMIT THE FILE CONTENTS TO GITHUB***
   kubectl apply -f argocd-repository-secret.yaml
   ```
   
   This creates secrets for Argo CD to access:
   - `hello-world-argocd-dev` repository (to read Application manifests)
   - `hello-world-config-dev` repository (to read Helm charts and values)

2. **Configure Argo CD access to GitHub Container Registry (GHCR)** (if not already done):
   ```bash
   # Edit ghcr-credentials-secret.yaml with your GitHub Argo CD machine username, token, email, and base64 credentials
   # ***BE SURE TO DELETE THE FILE AFTERWARDS SO AS NOT TO COMMIT THE FILE CONTENTS TO GITHUB***
   kubectl apply -f ghcr-credentials-secret.yaml
   ```
   
   **Note**: This secret can be set once for the entire GitHub organization and will be used by Argo CD to pull container images from GHCR for all environments. You do not need to create separate secrets for each environment repository.

3. **Apply the root Application (app-of-apps.yaml)**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/essesseff-hello-world-go-template/hello-world-argocd-dev/main/app-of-apps.yaml
   ```
   
   This root Application watches the `argocd/` directory in this repository and automatically applies `hello-world-dev-application.yaml`. Once applied, any changes to `argocd/hello-world-dev-application.yaml` will be automatically synced by Argo CD. Changes to other files in the root directory (README.md, etc.) will not trigger re-syncs.

4. **Verify in Argo CD UI**:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # Access: https://localhost:8080
   ```
   
   You should see:
   - `hello-world-argocd-dev` - Root Application (watches this repository)
   - `hello-world-dev` - Environment Application (auto-synced by root Application)

5. **Access the deployed application**:
   ```bash
   kubectl port-forward service/hello-world-dev 8081:80 -n essesseff-hello-world-go-template
   # Access: http://localhost:8081
   ```

## Application Details

- **Name**: `hello-world-dev`
- **Namespace**: `argocd`
- **Source Repository**: `hello-world-config-dev`
- **Destination Namespace**: `essesseff-hello-world-go-template`
- **Sync Policy**: Automated with prune and self-heal enabled

## Deployment Process

### Automatic Deployment

1. Push code to `main` branch in `hello-world` repository
2. GitHub Actions builds container image
3. essesseff webhook triggers
4. essesseff auto-updates `hello-world-config-dev/Chart.yaml` and `hello-world-config-dev/values.yaml` with new image tag
5. Argo CD syncs DEV Application automatically

## Repository URLs

- **Source**: `https://github.com/essesseff-hello-world-go-template/hello-world`
- **Config DEV**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-dev`
- **Argo CD DEV**: `https://github.com/essesseff-hello-world-go-template/hello-world-argocd-dev` (this repo)

## essesseff Integration

This setup requires the essesseff platform for deployment orchestration:

- **Event-driven promotions**: Automatic DEV deployments on code push
- **RBAC enforcement**: Role-based access control for deployments
- **Audit trail**: Complete history of all deployments

## Argo CD Configuration

### Reduce Git Polling Interval (Optional)

By default, Argo CD polls Git repositories every ~3 minutes (120-180 seconds). To reduce this to 60 seconds for faster change detection:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"60s","timeout.reconciliation.jitter":"10s"}}'
```

This will:
- Set base polling interval to 60 seconds
- Add up to 10 seconds of jitter (total: 60-70 seconds)
- Allow Argo CD to detect changes in `argocd/hello-world-dev-application.yaml` more quickly

**Note**: For even faster detection, consider configuring webhooks from GitHub to Argo CD for near-instant change detection.

## How It Works

1. **essesseff manages** image lifecycle and promotion decisions
2. **essesseff updates** `values.yaml` files in config repos (e.g., `hello-world-config-dev/values.yaml`) with approved image tags
3. **Argo CD detects** changes via Git polling (default: ~3 minutes, configurable to 60 seconds)
4. **Argo CD syncs** Application automatically (auto-sync enabled)
5. **Kubernetes resources** are updated with new image versions

## See Also

- [essesseff Documentation](https://essesseff.com/docs) - essesseff platform documentation
- [Argo CD Documentation](https://argo-cd.readthedocs.io/) - Argo CD documentation

