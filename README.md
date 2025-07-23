# Rundeck Internal ALB Migration Demo

## Overview
This demo shows how to migrate Rundeck from an internet-facing AWS ALB to an internal ALB, update environment variables, and trigger Rundeck jobs from GitHub Actions using a self-hosted runner.

---

## 1. Kubernetes Ingress: Internal ALB
- File: `ingress-internal.yaml`
- **Key Change:**
  - Annotation `alb.ingress.kubernetes.io/scheme` is set to `internal`.
  - Host is set to an internal DNS name (e.g., `rundeck-prod.internal.trailways.com`).
  - TLS is enforced with a Kubernetes TLS secret.
- **Checklist:**
  - [ ] `alb.ingress.kubernetes.io/certificate-arn`: Use your real ACM certificate ARN.
  - [ ] `host`: Use your internal DNS name and update DNS records as needed.
  - [ ] `tls.secretName`: Must match the TLS secret you create (see below).
  - [ ] `service.name` and `service.port.number`: Must match your Service.

---

## 2. Rundeck Deployment: Internal URL
- File: `deployment-internal.yaml`
- **Key Change:**
  - The `RUNDECK_GRAILS_URL` environment variable is set to the internal ALB DNS name.
  - Uses a fixed, stable Rundeck image tag.
  - All database credentials are pulled from Kubernetes secrets.
  - Persistent storage is mounted for Rundeck data.
  - Security context and resource limits are set.
  - Service account is specified for RBAC.
- **Checklist:**
  - [ ] `image`: Use a specific, tested Rundeck version (not `latest`).
  - [ ] `RUNDECK_GRAILS_URL`: Must match your internal DNS.
  - [ ] `RUNDECK_DATABASE_*`: Secret names/keys must match your DB secret.
  - [ ] `volumeMounts`/`volumes`: PVC name must match your actual PVC.
  - [ ] `resources`: Adjust CPU/memory as needed.
  - [ ] `serviceAccountName`: Use a service account with least privilege.
  - [ ] `replicas`: Set to 2+ for high availability.

---

## 3. Service
- File: `service.yaml`
- **Checklist:**
  - [ ] `name`: Must match the name used in Ingress.
  - [ ] `ports.port`: Must match Ingress backend port.
  - [ ] `ports.targetPort`: Must match container port.
  - [ ] `selector.app`: Must match Deployment label.

---

## 4. PersistentVolumeClaim
- File: `rundeck-data-pvc.yaml`
- **Checklist:**
  - [ ] `storageClassName`: Use your cluster’s storage class.
  - [ ] `resources.requests.storage`: Set size as needed.

---

## 5. Database Secret
- File: `rundeck-db-secret.yaml`
- **Checklist:**
  - [ ] `type`: Use your actual DB driver.
  - [ ] `jdbc`: Use your actual JDBC URL.
  - [ ] `user`: Your DB username.
  - [ ] `password`: Your DB password.

---

## 6. TLS Secret
- File: `rundeck-tls-secret.yaml`
- **Checklist:**
  - [ ] `tls.crt`/`tls.key`: Use your base64-encoded cert and key.
  - [ ] `secretName`: Must match the name in Ingress `tls.secretName`.
  - [ ] Use `kubectl create secret tls ...` for correct formatting.

---

## 7. Other Production Requirements
- [ ] **RBAC:** Ensure your service account has correct permissions.
- [ ] **Network Policies:** Restrict access to Rundeck as needed.
- [ ] **Monitoring/Logging:** Integrate with your monitoring/logging stack.
- [ ] **Backups:** Ensure persistent data and DB are backed up.
- [ ] **Self-hosted GitHub Runner:** Must be in the same VPC/subnet to trigger Rundeck jobs.
- [ ] **DNS:** Internal DNS must resolve to your internal ALB.
- [ ] **Certificates:** ACM and Kubernetes TLS secrets must match your internal DNS.

---

## 8. GitHub Actions: Self-Hosted Runner Required
Your GitHub Actions workflow must use a self-hosted runner inside your VPC to trigger Rundeck jobs via the internal ALB.

**Example:**
```yaml
name: Trigger Rundeck Job

on:
  workflow_dispatch:

jobs:
  trigger-rundeck:
    runs-on: [self-hosted, linux]
    steps:
      - name: Trigger Rundeck Job
        run: |
          curl -X POST "https://rundeck-prod.internal.trailways.com/api/36/job/JOB_ID/run" \
            -H "X-Rundeck-Auth-Token: ${{ secrets.RUNDECK_TOKEN }}"
```
- Replace `JOB_ID` and ensure your API token is in GitHub Secrets.

---

## 9. References
- [AWS ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/)
- [Rundeck API Docs](https://docs.rundeck.com/docs/api/)
- [GitHub Actions Self-hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) 