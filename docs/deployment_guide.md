# Deployment & Release Guide

## 1. Versioning Strategy
The project follows **Semantic Versioning (SemVer)**:
-   **MAJOR**: Architectural shifts (e.g., transition to 3-tier, new broker router).
-   **MINOR**: New strategies, additional ML models, or significant dashboard features.
-   **PATCH**: Bug fixes, indicator tweaks, and minor logic adjustments.

## 2. Release Workflow
1.  **Tagging**: When a milestone is reached, a new tag is created: `git tag -a v2.1.0 -m "Release description"`.
2.  **GitHub Release**: A release is drafted on GitHub, which triggers the `docker-autopush.yml` workflow to create immutable images.
3.  **VM Deployment**: The `deploy-to-vm.yml` workflow is manually triggered or automated on tag creation.

## 3. Deployment Steps (Manual & Automated)

### 3.1 Automated (GitHub Actions)
1.  **Code Sync**: `rsync` mirrors the repository to `/home/deploy/trading-bot` on the Hetzner VM.
2.  **Environment Check**: Script verifies `.env` existence and file permissions.
3.  **Container Update**: `docker-compose up -d --build` is executed via SSH to refresh the operational stack.
4.  **Health Verification**: Automated `curl` checks ensure the Webserver and Streamlit services are up.

### 3.2 Manual Deployment (Emergency)
If the CI/CD pipeline fails, use the `deploy_compose.sh` script located in the root:
```bash
./deploy_compose.sh --rebuild
```
This script handles container cleanup and sequential service startup.

## 5. Scheduling & Contingency
-   **Master Scheduler**: GitHub Actions (`scheduled-orchestrator.yml`) is the primary driver for all market sessions.
-   **VM Fallback**: The VM maintains an active `crontab` that mirrors the GHA schedule.
-   **GHA-Check Logic**: The VM orchestrator utilizes a 3-hour data freshness check. If GHA successfully syncs fresh data, the VM's cron job terminates early (detecting the data is fresh). If GHA fails, the VM cron job detects stale data and automatically begins local harvesting as a fallback.
-   **Timezone**: The VM is fixed to `UTC/GMT` to unify scheduling across GHA and production.

## 6. Rollback Procedures
-   **Automated Rollback**: Redeploy the previous Git tag via the GitHub Actions UI.
-   **Manual Rollback**:
    1.  On the VM, navigate to the backup directory: `cd /home/deploy/trading-bot/backups`.
    2.  Locate the previous stable daily snapshot.
    3.  Run `git checkout <tag_id>`.
    4.  Restart services: `docker compose up -d`.

## 5. Environment Configuration
-   **VM Spec**: Ubuntu 22.04 LTS, 4 vCPUs, 8GB RAM.
-   **Storage**: 80GB Block Storage for logs and ML databases.
-   **Networking**: Inbound traffic restricted to port 10000 (Monitoring) and 8501 (Dashboard) with IP whitelisting.
-   **Timezone**: Fixed to `America/New_York` to align with NYSE market hours.
