# MongoDB Log Management CronJobs

This directory contains Kubernetes CronJob manifests for managing MongoDB service logs stored on NFS.

## Overview

These CronJobs replace the traditional cron entries that were previously configured as:

```bash
0 0 * * * find /app/milky-way/logs/service/MONGO -mtime +1 -type f -name "*.log*" -exec gzip '{}' \;
0 0 * * * find /app/milky-way/logs/service/MONGO -mtime +7 -type f -name "*.gz" -exec rm -f '{}' \;
30 1 * * * find /app/milky-way/logs/service/MONGO -mtime +30 -type f -exec rm -f '{}' \;
```

## CronJobs

### 1. Log Compression (`log-compression-cronjob.yaml`)

- **Schedule**: Daily at midnight (0 0 * * *)
- **Purpose**: Compresses log files older than 1 day
- **Action**: Finds `.log*` files (excluding already compressed `.gz` files) older than 1 day and compresses them with gzip

### 2. Compressed Log Cleanup (`compressed-log-cleanup-cronjob.yaml`)

- **Schedule**: Daily at 12:30 AM (30 0 * * *)
- **Purpose**: Removes old compressed log files
- **Action**: Deletes `.gz` files older than 7 days
- **Note**: Runs 30 minutes after compression to avoid race conditions

### 3. General Log Cleanup (`general-log-cleanup-cronjob.yaml`)

- **Schedule**: Daily at 1:30 AM (30 1 * * *)
- **Purpose**: Safety cleanup for very old log files
- **Action**: Deletes all files older than 30 days (catches any files missed by other jobs)

## NFS Configuration

All CronJobs mount the same NFS volume:

- **Server**: `sila-ipdc.warmipdc.intra.ais`
- **Path**: `/FS_MILKYWAY_LOG/legacy-portal-apis/logs/service/MONGO`
- **Mount Point**: `/logs` (inside the container)

## Deployment

To apply these CronJobs to your Kubernetes cluster:

```bash
# Apply all CronJobs
kubectl apply -f k8s/cronjobs/

# Or apply individually
kubectl apply -f k8s/cronjobs/log-compression-cronjob.yaml
kubectl apply -f k8s/cronjobs/compressed-log-cleanup-cronjob.yaml
kubectl apply -f k8s/cronjobs/general-log-cleanup-cronjob.yaml
```

## Verification

Check the status of the CronJobs:

```bash
# List all CronJobs in the banraiphisan namespace
kubectl get cronjobs -n banraiphisan

# View details of a specific CronJob
kubectl describe cronjob mongo-log-compression -n banraiphisan

# View logs of the most recent job run
kubectl logs -n banraiphisan -l job-name=<job-name>
```

## Monitoring

The CronJobs are configured to:

- Keep the last 3 successful job histories
- Keep the last 3 failed job histories
- Prevent concurrent executions (`concurrencyPolicy: Forbid`)

## Resource Limits

Each CronJob container is configured with:

- **Requests**: 100m CPU, 128Mi memory
- **Limits**: 500m CPU, 256Mi memory

These can be adjusted based on the actual log volume and performance requirements.

## Notes

- All CronJobs use the `alpine:latest` image with necessary utilities installed at runtime
- The NFS server must be accessible from the Kubernetes cluster
- Ensure the namespace `banraiphisan` exists before deploying
- The CronJobs will run with the default service account unless specified otherwise
