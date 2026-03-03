title: DevOps Alchemist
description: Transforms complex deployment processes into seamless workflows
version: 1.0.0
tags: [automation, CI/CD, deployment, optimization, DevOps]
author: OpenClaw Team
dependencies:
  - docker >= 20.10
  - kubectl >= 1.25
  - helm >= 3.12
  - terraform >= 1.5
  - ansible >= 2.14
  - git
  - jq
  - curl
required_env:
  - DOCKER_REGISTRY
  - K8S_CLUSTER_ENDPOINT
  - TERRAFORM_VAR_FILE (optional)
---

# DevOps Alchemist

## Purpose

Transforms complex deployment processes into seamless, reliable, and observable workflows. Specializes in:
- **Infrastructure as Code**: Terraform plan/apply with state management and drift detection
- **Container Orchestration**: Zero-downtime Kubernetes deployments with rollout monitoring
- **CI/CD Pipeline Optimization**: GitHub Actions/Docker Hub/ArgoCD integration and caching strategies
- **Configuration Management**: Ansible playbooks for multi-environment consistency
- **Observability & Health Checks**: Prometheus metrics, log aggregation, and automated rollback triggers
- **Secret Management**: Integration with Vault/Sealed Secrets/SSM Parameter Store

Real-world use cases:
- "Deploy microservices to production with canary release and automatic rollback on 5xx errors"
- "Optimize Docker build cache to reduce CI pipeline time by 40%"
- "Implement blue-green deployment for legacy monolithic app with database migration safety"
- "Configure GitOps workflow with ArgoCD and automated PR generation"
- "Set up infrastructure drift detection and remediation across 3 AWS regions"

## Scope

### Infrastructure Provisioning
```bash
terraform init -backend-config="bucket=my-terraform-state" -reconfigure
terraform plan -var-file=env/prod.tfvars -out=tfplan
terraform apply tfplan
terraform destroy -var-file=env/prod.tfvars -auto-approve
```

### Kubernetes Operations
```bash
kubectl apply -f k8s/manifests/ -n production --server-side
kubectl set image deployment/myapp myapp=$IMAGE_TAG --record
kubectl rollout status deployment/myapp -n production --timeout=5m
kubectl rollout undo deployment/myapp -n production --to-revision=3
helm upgrade --install myapp ./helm-chart -f values/prod.yaml --wait --timeout 5m
```

### CI/CD Pipeline
```bash
docker build --build-arg NODE_ENV=production -t $DOCKER_REGISTRY/myapp:$SHA .
docker push $DOCKER_REGISTRY/myapp:$SHA
docker scan --severity high $DOCKER_REGISTRY/myapp:$SHA
```

### Configuration Management
```bash
ansible-playbook -i inventories/production.yml playbooks/deploy.yml --extra-vars "image_tag=$SHA" --limit web-01,web-02
```

### GitOps Sync
```bash
argocd app sync myapp-production --prune --retry-limit 5 --timeout 10m
argocd app wait myapp-production --timeout 10m
argocd app history myapp-production
```

## Work Process

1. **Assessment & Context**
   - Read current deployment manifests, Terraform state, and CI/CD config
   - Check infrastructure drift: `terraform plan` (non-destructive)
   - Verify cluster health: `kubectl get nodes`, `kubectl cluster-info`
   - Examine recent deployment history: `kubectl get rs --sort-by=.metadata.creationTimestamp`

2. **Planning Phase**
   - Generate Terraform plan if infrastructure changes: `terraform plan -detailed-exitcode`
   - Validate Kubernetes manifests: `kubectl apply --dry-run=client -f manifests/`
   - Check Helm dependencies: `helm dependency update ./chart && helm lint ./chart`
   - Review CI/CD cache strategy: Analyze `.github/workflows/*.yml` for caching patterns

3. **Execution Phase**
   - Lock state if needed: `terraform force-unlock <lock_id>` only after confirming no active apply
   - Perform rolling update with verification:
     ```bash
     kubectl set image deployment/$APP $CONTAINER=$IMAGE --record
     kubectl rollout status deployment/$APP --timeout=6m
     kubectl get pods -l app=$APP -o wide
     ```
   - Monitor health endpoints: `curl -f http://$SERVICE/health || rollback`
   - Run post-deployment smoke tests using `curl` or `httpie`

4. **Verification & Observability**
   - Check pod status: `kubectl get pods -n $NAMESPACE --field-selector=status.phase!=Running`
   - Verify deployment metrics: `kubectl top pods -n $NAMESPACE --containers`
   - Inspect logs for errors: `kubectl logs -n $NAMESPACE --tail=100 --since=5m deployment/$APP`
   - Test external connectivity: ` curl -s -o /dev/null -w "%{http_code}" https://$DOMAIN/healthz`
   - Confirm metrics: `curl -s http://prometheus:9090/api/v1/query?query=up{job="$APP"}`

5. **Cleanup & Documentation**
   - Remove unused resources: ` kubectl delete hpa,rs --field-selector=metadata.creationTimestamp<$(date -d "24 hours ago" -Is)`
   - Compact Docker images: `docker image prune -af --filter "until=24h"`
   - Prune Terraform state: Archive old state versions, run `terraform state replace-provider` if needed

## Golden Rules

1. **Zero-Trust Rollbacks**: Always test rollback command in dry-run mode first:
   ```bash
   kubectl rollout undo deployment/$APP --dry-run=client
   terraform plan -destroy -target=module.$MODULE
   ```

2. **State Safety**: Never run `terraform apply` without a saved plan file. Verify plan exit code:
   - 0: no changes
   - 1: error
   - 2: changes exist (must review)

3. **Canary First**: For production deployments, use `kubectl rollout pause` after first replica, validate, then `kubectl rollout resume`

4. **Observability Gates**: Deployment success requires:
   - All pods in Ready state (not just Running)
   - Health endpoint returning 200
   - Error rate < 1% for 5 minutes (check metrics)
   - Latency p99 within baseline threshold

5. **Secret Sanitization**: Never expose `$DOCKER_REGISTRY_PASSWORD`, `$KUBECONFIG`, or Terraform vars in logs. Use `--quiet` flags and masking.

6. **Capacity Pre-check**: Before scaling, verify cluster resources:
   ```bash
   kubectl describe nodes | grep -A 5 Allocatable
   kubectl top nodes
   ```

## Examples

### Example 1: Deploy New Version with Canary
User prompt: "Deploy v2.3.0 to production with 10% canary and auto-rollback on errors"

DevOps Alchemist actions:
```bash
# Check current deployment
kubectl get deploy myapp -n prod -o jsonpath='{.spec.replicas}'
# 10

# Set image with pause at 1 replica
kubectl set image deploy/myapp myapp=myrepo/myapp:v2.3.0 -n prod --record
kubectl rollout pause deploy/myapp -n prod
# Wait for first pod to be ready
kubectl rollout status deploy/myapp -n prod --timeout=3m
# Validate canary pod
CANARY_POD=$(kubectl get pods -n prod -l app=myapp --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
curl -f http://$CANARY_POD:8080/health || kubectl rollout undo deploy/myapp -n prod
# If healthy, resume to full rollout
kubectl rollout resume deploy/myapp -n prod
kubectl rollout status deploy/myapp -n prod --timeout=5m
```

### Example 2: Terraform Drift Remediation
User prompt: "Check for infrastructure drift in prod and apply if safe"

DevOps Alchemist actions:
```bash
cd terraform/environments/prod
terraform init -backend-config="bucket=prod-state" -reconfigure
terraform plan -detailed-exitcode -out=drift-check
EXIT_CODE=$?
if [ $EXIT_CODE -eq 2 ]; then
  echo "Drift detected. Showing changes:"
  terraform show -json drift-check | jq '.resource_changes[] | {address, actions, before, after}'
  # User confirmation required for apply
  read -p "Apply changes? (yes/no): " CONFIRM
  if [ "$CONFIRM" = "yes" ]; then
    terraform apply drift-check
  fi
else
  echo "No drift detected or error occurred."
fi
```

### Example 3: Optimize Docker Build Cache
User prompt: "Optimize Dockerfile to leverage cache and reduce build time"

DevOps Alchemist actions:
```bash
docker build --target builder -t myapp:builder .
docker build --cache-from myapp:builder -t myapp:latest .
docker images --filter=reference='myapp:*' --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
# Suggest Dockerfile changes: order layers from least to most frequently changed
```

### Example 4: Emergency Rollback
User prompt: "Rollback myapp production to previous version now"

DevOps Alchemist actions:
```bash
# Check revision history
kubectl rollout history deployment/myapp -n production
# Revision 4: update image to "myrepo/myapp:v2.3.0" (1h ago)
# Revision 3: update image to "myrepo/myapp:v2.2.1" (3d ago)

# Perform rollback to revision 3
kubectl rollout undo deployment/myapp -n production --to-revision=3
kubectl rollout status deployment/myapp -n production --timeout=6m
# Verify pods
kubectl get pods -n production -l app=myapp -o wide
```

### Example 5: CI/CD Pipeline Debugging
User prompt: "CI pipeline fails at Docker push with 'denied: requested access to the resource is denied'"

DevOps Alchemist actions:
```bash
# Check GitHub Secrets
# Ensure DOCKER_REGISTRY_USERNAME and DOCKER_REGISTRY_PASSWORD are set
# Validate Docker login in CI step:
echo $DOCKER_REGISTRY_PASSWORD | docker login $DOCKER_REGISTRY -u $DOCKER_REGISTRY_USERNAME --password-stdin
docker push $DOCKER_REGISTRY/myapp:$GITHUB_SHA
# If using GitHub Container Registry, ensure token has 'write:packages' scope
```

## Rollback Commands

### Kubernetes Deployment Rollback
```bash
# Rollback to previous revision
kubectl rollout undo deployment/<name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=<N>

# Check rollback status
kubectl rollout status deployment/<name> -n <namespace> --timeout=5m

# Abort ongoing rollout
kubectl rollout pause deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>

# Verify rollback by comparing images
kubectl get rs -n <namespace> --sort-by=.metadata.creationTimestamp | tail -2
kubectl describe rs <old-rs-name> -n <namespace> | grep Image
```

### Terraform Rollback
```bash
# List state versions (requires remote backend)
terraform state list

# Rollback to specific state version (if using S3 backend with versioning)
aws s3 cp s3://my-terraform-state/<env>/terraform.tfstate.backup terraform.tfstate.rollback
terraform apply terraform.tfstate.rollback

# Target rollback of specific resource
terraform apply -target=module.database -var-file=env/prod.tfvars

# Force destroy problematic resource (last resort)
terraform state rm module.broken_resource
terraform apply -var-file=env/prod.tfvars
```

### Docker Image Rollback
```bash
# List available tags
docker images myrepo/myapp

# Pull previous version
docker pull myrepo/myapp:v2.2.1

# Trigger new deployment with old image (K8s)
kubectl set image deployment/myapp myapp=myrepo/myapp:v2.2.1 -n production
kubectl rollout status deployment/myapp -n production

# Remove broken tag locally (optional)
docker rmi myrepo/myapp:latest
docker tag myrepo/myapp:v2.2.1 myrepo/myapp:latest
```

### Helm Rollback
```bash
# List release history
helm history myapp -n production

# Rollback to revision 3
helm rollback myapp 3 -n production

# Verify rollback
helm status myapp -n production
kubectl get pods -l app.kubernetes.io/instance=myapp -n production
```

### Ansible Rollback
```bash
# If using tags, run playbook with previous configuration
ansible-playbook -i inventories/production.yml playbooks/rollback.yml \
  --extra-vars "previous_image_tag=v2.2.1" \
  --limit web-*

# Manual rollback via direct command on servers (emergency)
ansible all -i inventories/production.yml -m shell -a "docker stop myapp && docker rm myapp && docker run -d --name myapp myrepo/myapp:v2.2.1" --limit web-*
```

## Dependencies & Requirements

- **Docker**: Build, scan, and push container images. Requires login to registry.
- **kubectl**: Configured with kubeconfig for target cluster (check `$KUBECONFIG`).
- **helm**: For Helm chart deployments. Initialize local repo if needed.
- **terraform**: Remote backend (S3/GCS) with state locking via DynamoDB.
- **ansible**: SSH access to target hosts; inventory file present.
- **jq**: JSON parsing for API responses and Terraform output.
- **curl**: Health checks and API testing.
- **Git**: Clone/pull repositories, manage branches.

Ensure tools are in PATH and properly versioned (`docker version`, `kubectl version --short`, `terraform version`).

## Verification Steps

1. **Post-deployment smoke test**:
   ```bash
   curl -f -s -o /dev/null -w "%{http_code}" https://$SERVICE/health || exit 1
   ```

2. **Pod readiness**:
   ```bash
   NOT_READY=$(kubectl get pods -n $NS -l app=$APP --no-headers | awk '$4!=0 {print $1}')
   if [ -n "$NOT_READY" ]; then
     echo "Pods not ready: $NOT_READY"
     exit 1
   fi
   ```

3. **Resource utilization sanity**:
   ```bash
   kubectl top pods -n $NS --containers | grep -E "(high|warning)" || echo "All good"
   ```

4. **Log error detection**:
   ```bash
   ERRORS=$(kubectl logs -n $NS --since=2m deployment/$APP 2>&1 | grep -iE "error|exception|failed" | wc -l)
   [ $ERRORS -eq 0 ] || echo "Found $ERRORS error lines in logs"
   ```

5. **External connectivity**:
   ```bash
   nc -zv $DOMAIN 443 || echo "Cannot connect to $DOMAIN:443"
   ```

## Troubleshooting

### Issue: `kubectl apply` hangs or times out
- Cause: API server overload or network partition
- Fix: Check cluster status: `kubectl get cs`. Increase timeout: `--request-timeout=30s`. Retry with `--validate=false`.

### Issue: Terraform plan shows "Error: Provider configuration not present"
- Fix: Reinitialize backend: `terraform init -reconfigure`. Verify backend credentials in `backend.tf`.

### Issue: Docker build cache invalidated unexpectedly
- Fix: Check Dockerfile layer order. Ensure `COPY package*.json` comes before `COPY src/`. Use `--target` for multi-stage builds.

### Issue: Helm release stuck in "pending-upgrade"
- Fix: Check another plugin: `helm ls -A`. Delete failed release: `helm uninstall <release> --no-hooks`. Retry with `--force --atomic`.

### Issue: Ansible SSH connection refused
- Fix: Verify SSH key permissions: `chmod 600 ~/.ssh/id_rsa`. Ensure `ansible_user` in inventory. Test connectivity: `ansible all -m ping -i inventory.yml`.

### Issue: CI pipeline cache miss leading to slow builds
- Fix: Cache Docker layers: `docker buildx build --cache-from type=registry,ref=$IMAGE --cache-to type=inline`. Alternatively, use GitHub Actions `actions/cache` for `~/.docker`.

### Issue: Post-deployment 5xx errors spike
- Immediate: `kubectl rollout undo deployment/$APP`
- Investigation: `kubectl logs -p deployment/$APP` (previous container logs). Check DB connection pool exhaustion. Verify env vars and secrets.

### Issue: Terraform state drift after manual change
- Fix: Import resource: `terraform import module.resource aws_resource_id`. Or run `terraform apply` to adopt changes if intentional.

### Issue: Pods stuck in `CrashLoopBackOff`
- Investigate: `kubectl logs <pod> --previous`. Common causes: OOMKilled (`kubectl describe pod`), missing configmap/secret, init container failure.

### Issue: Helm chart dependency fetch fails
- Fix: `helm dependency update ./chart --skip-refresh`. Ensure `repositories` in `Chart.yaml` are reachable. Clear Helm cache: `helm repo rm <repo> && helm repo add <repo> <url>`.

---
This skill automates deployment with safety gates, observability integration, and explicit rollback pathways. All commands assume standard tooling and proper prior configuration.