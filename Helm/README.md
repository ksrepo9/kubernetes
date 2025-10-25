# Helm Scenario-Based Interview Questions and Answers


## Table of Contents
- [Scenario 1: Helm Chart Structure](#scenario-1-helm-chart-structure)
- [Scenario 2: Values Management](#scenario-2-values-management)
- [Scenario 3: Helm Release Troubleshooting](#scenario-3-helm-release-troubleshooting)
- [Scenario 4: Custom Chart Development](#scenario-4-custom-chart-development)
- [Scenario 5: Helm Hooks](#scenario-5-helm-hooks)
- [Scenario 6: Dependency Management](#scenario-6-dependency-management)
- [Scenario 7: Security & Secrets](#scenario-7-security--secrets)
- [Scenario 8: Upgrade & Rollback Strategies](#scenario-8-upgrade--rollback-strategies)
- [Scenario 9: Multi-Environment Deployment](#scenario-9-multi-environment-deployment)
- [Scenario 10: CI/CD Integration](#scenario-10-cicd-integration)

---

## Scenario 1: Helm Chart Structure

**Question**: You're asked to review a Helm chart. What key components would you check to ensure it follows best practices?

**Answer**:

### Essential Chart Components:
```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration
├── charts/             # Chart dependencies
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl    # Template helpers
└── crds/               # Custom Resource Definitions
```

### Chart.yaml Best Practices:
```yaml
apiVersion: v2
name: my-application
description: A Helm chart for Kubernetes
type: application
version: 1.2.3
appVersion: "v2.1.0"
dependencies:
  - name: redis
    version: "14.0.0"
    repository: "https://charts.bitnami.com/bitnami"
keywords:
  - web
  - application
  - microservice
maintainers:
  - name: John Doe
    email: john@example.com
```

### Template Validation:
```bash
# Lint the chart
helm lint my-chart/

# Dry run to see rendered templates
helm install my-release ./my-chart --dry-run --debug

# Check Kubernetes manifest syntax
helm template my-release ./my-chart | kubeval --strict
```

---

## Scenario 2: Values Management

**Question**: How would you manage different configurations for development, staging, and production environments using Helm?

**Answer**:

### Values File Structure:
```
environments/
├── values-dev.yaml
├── values-staging.yaml
└── values-prod.yaml
```

### Development Values (values-dev.yaml):
```yaml
replicaCount: 1
image:
  repository: my-app
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

ingress:
  enabled: false

autoscaling:
  enabled: false
```

### Production Values (values-prod.yaml):
```yaml
replicaCount: 3
image:
  repository: my-app
  tag: "v1.2.3"
  pullPolicy: Always

resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.company.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Deployment Commands:
```bash
# Development deployment
helm install my-app ./my-chart -f values-dev.yaml -n development

# Production deployment
helm upgrade --install my-app ./my-chart -f values-prod.yaml -n production
```

---

## Scenario 3: Helm Release Troubleshooting

**Question**: A Helm upgrade failed and the release is stuck. How would you troubleshoot and recover?

**Answer**:

### Troubleshooting Steps:
```bash
# Check release status
helm status my-release
helm get manifest my-release
helm get hooks my-release

# Check revision history
helm history my-release

# See what changed between revisions
helm get manifest my-release --revision=2
helm get manifest my-release --revision=3

# Check Kubernetes resources
kubectl get all -l app.kubernetes.io/instance=my-release

# Examine failed pods
kubectl describe pod -l app.kubernetes.io/instance=my-release
kubectl logs -l app.kubernetes.io/instance=my-release
```

### Recovery Strategies:
```bash
# Rollback to previous version
helm rollback my-release 2

# Force upgrade if necessary
helm upgrade my-release ./my-chart --force

# Uninstall if completely broken
helm uninstall my-release

# Check for hanging resources
kubectl get all --all-namespaces | grep my-release
```

### Common Issues and Fixes:
1. **ConfigMap/Secret immutable errors**:
   ```bash
   # Use --force to recreate resources
   helm upgrade my-release ./my-chart --force
   ```

2. **Resource name conflicts**:
   ```bash
   # Use atomic to ensure clean rollback on failure
   helm upgrade my-release ./my-chart --atomic
   ```

3. **Hook failures**:
   ```bash
   # Check hook status and manually cleanup if needed
   kubectl get jobs -l helm.sh/hook
   ```

---

## Scenario 4: Custom Chart Development

**Question**: You need to create a Helm chart for a microservice that requires dynamic configuration based on environment. How would you design the templates?

**Answer**:

### Template with Conditional Logic:
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-chart.fullname" . }}-config
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
data:
  app.properties: |
    {{- if .Values.production }}
    environment=production
    log.level=WARN
    database.pool.size=20
    {{- else }}
    environment=development
    log.level=DEBUG
    database.pool.size=5
    {{- end }}
    server.port=8080
    app.name={{ .Values.app.name }}
```

### Advanced Template Functions:
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
    {{- with .Values.deployment.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- if .Values.env }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Helpers Template (_helpers.tpl):
```tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "my-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-chart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-chart.labels" -}}
helm.sh/chart: {{ include "my-chart.chart" . }}
{{ include "my-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

## Scenario 5: Helm Hooks

**Question**: You need to run database migrations before deploying a new version of your application. How would you implement this using Helm hooks?

**Answer**:

### Database Migration Hook:
```yaml
# templates/job-db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-chart.fullname" . }}-db-migrate
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    metadata:
      name: {{ include "my-chart.fullname" . }}-db-migrate
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["/bin/sh", "-c"]
        args:
          - |
            ./bin/migrate \
            --database-url={{ .Values.database.url }} \
            --command=up
        env:
          - name: DATABASE_URL
            valueFrom:
              secretKeyRef:
                name: {{ .Values.database.secretName }}
                key: url
        resources:
          {{- toYaml .Values.migration.resources | nindent 12 }}
```

### Data Backup Hook (Pre-Upgrade):
```yaml
# templates/job-pre-backup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-chart.fullname" . }}-pre-backup
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:13
        command: ["/bin/sh", "-c"]
        args:
          - |
            pg_dump -h ${DB_HOST} -U ${DB_USER} ${DB_NAME} > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
        env:
          - name: DB_HOST
            value: {{ .Values.database.host }}
          - name: DB_USER
            valueFrom:
              secretKeyRef:
                name: {{ .Values.database.secretName }}
                key: username
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: {{ .Values.database.secretName }}
                key: password
          - name: DB_NAME
            value: {{ .Values.database.name }}
        volumeMounts:
        - name: backup-volume
          mountPath: /backup
      volumes:
      - name: backup-volume
        persistentVolumeClaim:
          claimName: {{ .Values.backup.pvcName }}
      restartPolicy: OnFailure
```

### Hook Weight System:
- **pre-install**: -10 (backup), -5 (migration), 0 (main resources)
- **post-install**: 5 (smoke tests), 10 (notifications)
- **pre-upgrade**: -10 (backup), -5 (migration)
- **post-upgrade**: 5 (verification)

---

## Scenario 6: Dependency Management

**Question**: Your application depends on Redis and PostgreSQL. How would you manage these dependencies in your Helm chart?

**Answer**:

### Chart.yaml Dependencies:
```yaml
apiVersion: v2
name: my-application
description: A Helm chart for Kubernetes
type: application
version: 1.2.3
appVersion: "v2.1.0"

dependencies:
  - name: redis
    version: "17.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - database
      - cache
  - name: postgresql
    version: "12.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database
```

### Values.yaml Dependency Configuration:
```yaml
# Redis dependency configuration
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    password: "redis-password"
  master:
    persistence:
      enabled: true
      size: 8Gi

# PostgreSQL dependency configuration
postgresql:
  enabled: true
  auth:
    postgresPassword: "postgres-password"
    database: "myapp"
    username: "myapp"
    password: "myapp-password"
  primary:
    persistence:
      enabled: true
      size: 20Gi

# Application configuration
app:
  database:
    host: "my-application-postgresql"
    name: "myapp"
  redis:
    host: "my-application-redis-master"
```

### Dependency Management Commands:
```bash
# Download dependencies
helm dependency update

# List dependencies
helm dependency list

# Build dependencies charts directory
helm dependency build

# Package with dependencies
helm package . --dependency-update
```

### Conditional Dependency Templates:
```yaml
# templates/app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-chart.fullname" . }}-config
data:
  application.yml: |
    database:
      url: jdbc:postgresql://{{ .Values.postgresql.primary.service.name }}:5432/{{ .Values.postgresql.auth.database }}
      {{- if .Values.postgresql.enabled }}
      username: {{ .Values.postgresql.auth.username }}
      password: {{ .Values.postgresql.auth.password }}
      {{- end }}
    
    redis:
      host: {{ .Values.redis.master.service.name }}
      {{- if .Values.redis.enabled }}
      port: 6379
      password: {{ .Values.redis.auth.password }}
      {{- end }}
```

---

## Scenario 7: Security & Secrets

**Question**: How would you handle sensitive information like passwords and API keys in a Helm chart?

**Answer**:

### Approach 1: External Secrets Management
```yaml
# values-prod.yaml (NO SECRETS HERE)
database:
  secretName: "app-database-secret"
redis:
  secretName: "app-redis-secret"
api:
  secretName: "app-api-secret"
```

### Approach 2: Helm Secrets with SOPS
```bash
# Install helm-secrets plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt values file
sops --encrypt values-secrets.yaml > values-secrets.enc.yaml

# Install with decryption
helm secrets install my-app ./my-chart -f values.yaml -f values-secrets.enc.yaml
```

### Approach 3: Kubernetes Secrets Template
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "my-chart.fullname" . }}-secrets
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
type: Opaque
data:
  {{- if .Values.secrets.databasePassword }}
  database-password: {{ .Values.secrets.databasePassword | b64enc | quote }}
  {{- end }}
  {{- if .Values.secrets.redisPassword }}
  redis-password: {{ .Values.secrets.redisPassword | b64enc | quote }}
  {{- end }}
  {{- if .Values.secrets.apiKey }}
  api-key: {{ .Values.secrets.apiKey | b64enc | quote }}
  {{- end }}
```

### Approach 4: External Secrets Operator Integration
```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "my-chart.fullname" . }}-external-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore
  target:
    name: {{ include "my-chart.fullname" . }}-secrets
  data:
  - secretKey: database-password
    remoteRef:
      key: /apps/prod/database
      property: password
  - secretKey: api-key
    remoteRef:
      key: /apps/prod/api
      property: key
```

### Best Practices:
1. **Never commit secrets to version control**
2. **Use Helm --set for sensitive values in CI/CD**:
   ```bash
   helm upgrade --install my-app ./my-chart \
     --set secrets.databasePassword=$DB_PASSWORD \
     --set secrets.apiKey=$API_KEY
   ```
3. **Use secret management tools** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
4. **Rotate secrets regularly**

---

## Scenario 8: Upgrade & Rollback Strategies

**Question**: What strategies would you implement for safe Helm upgrades in production?

**Answer**:

### 1. Atomic Upgrades:
```bash
helm upgrade my-app ./my-chart \
  --atomic \
  --timeout 10m \
  --wait \
  --wait-for-jobs
```

### 2. Canary Deployment Strategy:
```yaml
# values-canary.yaml
replicaCount: 5
canary:
  enabled: true
  replicas: 1
  image:
    tag: "v2.0.0-canary"

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  {{- if .Values.canary.enabled }}
  annotations:
    rollme: {{ randAlphaNum 5 | quote }}
  {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.canary.enabled | ternary 1 2 }}
      maxUnavailable: 0
```

### 3. Blue-Green Deployment:
```bash
# Install new version as separate release
helm install my-app-v2 ./my-chart -f values-prod.yaml

# Test new version
kubectl get service my-app-v2-service

# Switch traffic (update main service selector)
kubectl patch service my-app-service -p '{"spec":{"selector":{"version":"v2"}}}'

# Uninstall old version
helm uninstall my-app-v1
```

### 4. Rollback Automation:
```bash
#!/bin/bash
# upgrade-with-rollback.sh

RELEASE_NAME="my-app"
CHART_PATH="./my-chart"
TIMEOUT="600s"

echo "Starting Helm upgrade..."
helm upgrade $RELEASE_NAME $CHART_PATH \
  --atomic \
  --timeout $TIMEOUT \
  --wait

if [ $? -ne 0 ]; then
    echo "Upgrade failed! Rolling back..."
    helm rollback $RELEASE_NAME
    exit 1
fi

echo "Upgrade successful!"
```

### 5. Health Checks Integration:
```yaml
# values.yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 30
```

---

## Scenario 9: Multi-Environment Deployment

**Question**: How would you manage Helm deployments across multiple environments (dev, staging, prod) with different configurations?

**Answer**:

### Directory Structure:
```
my-chart/
├── Chart.yaml
├── values.yaml          # Base values
├── values-dev.yaml      # Development overrides
├── values-staging.yaml  # Staging overrides
├── values-prod.yaml     # Production overrides
├── templates/
└── charts/
```

### Environment-Specific Values:

#### Base values.yaml:
```yaml
# Default values
replicaCount: 1
image:
  repository: my-app
  tag: latest
  pullPolicy: IfNotPresent

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts: []
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
```

#### Production values-prod.yaml:
```yaml
replicaCount: 3

image:
  tag: "v1.2.3"
  pullPolicy: Always

ingress:
  enabled: true
  className: "nginx"
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.company.com

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2
    memory: 4Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

database:
  host: "postgresql-prod.company.com"
  ssl: true
```

### Deployment Scripts:
```bash
#!/bin/bash
# deploy-to-env.sh

ENVIRONMENT=$1
VERSION=$2

case $ENVIRONMENT in
  dev)
    VALUES_FILE="values-dev.yaml"
    NAMESPACE="development"
    ;;
  staging)
    VALUES_FILE="values-staging.yaml"
    NAMESPACE="staging"
    ;;
  prod)
    VALUES_FILE="values-prod.yaml"
    NAMESPACE="production"
    ;;
  *)
    echo "Unknown environment: $ENVIRONMENT"
    exit 1
    ;;
esac

echo "Deploying to $ENVIRONMENT with version $VERSION"

helm upgrade --install my-app ./my-chart \
  --namespace $NAMESPACE \
  --create-namespace \
  --values $VALUES_FILE \
  --set image.tag=$VERSION \
  --atomic \
  --timeout 10m \
  --wait
```

### Kubernetes Context Management:
```bash
#!/bin/bash
# switch-context.sh

ENVIRONMENT=$1

case $ENVIRONMENT in
  dev)
    kubectl config use-context dev-cluster
    ;;
  staging)
    kubectl config use-context staging-cluster
    ;;
  prod)
    kubectl config use-context prod-cluster
    ;;
esac
```

---

## Scenario 10: CI/CD Integration

**Question**: How would you integrate Helm into a CI/CD pipeline with proper testing and validation?

**Answer**:

### GitLab CI Example:
```yaml
# .gitlab-ci.yml
stages:
  - test
  - package
  - deploy

variables:
  HELM_VERSION: "3.12.0"

.before_script:
  - curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  - chmod 700 get_helm.sh
  - ./get_helm.sh --version v${HELM_VERSION}

helm-lint:
  stage: test
  script:
    - helm lint ./chart
    - helm template my-app ./chart --debug

helm-test:
  stage: test
  script:
    - helm install my-app ./chart --dry-run --debug
    - helm test my-app --timeout 300s

package-chart:
  stage: package
  script:
    - helm package ./chart --version ${CI_COMMIT_TAG} --app-version ${CI_COMMIT_TAG}
    - helm repo index . --url https://charts.company.com
  artifacts:
    paths:
      - *.tgz
    expire_in: 1 week
  only:
    - tags

deploy-dev:
  stage: deploy
  script:
    - helm upgrade --install my-app ./chart
      --namespace development
      --values values-dev.yaml
      --set image.tag=${CI_COMMIT_SHA}
      --atomic
      --wait
  environment:
    name: development
  only:
    - main

deploy-staging:
  stage: deploy
  script:
    - helm upgrade --install my-app ./chart
      --namespace staging
      --values values-staging.yaml
      --set image.tag=${CI_COMMIT_TAG}
      --atomic
      --wait
  environment:
    name: staging
  only:
    - tags

deploy-prod:
  stage: deploy
  script:
    - helm upgrade --install my-app ./chart
      --namespace production
      --values values-prod.yaml
      --set image.tag=${CI_COMMIT_TAG}
      --atomic
      --wait
  environment:
    name: production
  when: manual
  only:
    - tags
```

### GitHub Actions Example:
```yaml
# .github/workflows/helm.yaml
name: Helm Deploy

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.12.0'
    - name: Run Helm Lint
      run: |
        helm lint ./chart
        helm template my-app ./chart --debug

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
    - uses: actions/checkout@v3
    - name: Set up Helm
      uses: azure/setup-helm@v3
    - name: Install kind
      uses: helm/kind-action@v1
    - name: Run Helm Tests
      run: |
        helm dependency update ./chart
        helm install my-app ./chart --wait
        helm test my-app

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    environment: 
      name: ${{ github.ref == 'refs/heads/main' && 'staging' || 'production' }}
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
    - uses: actions/checkout@v3
    - name: Set up Helm
      uses: azure/setup-helm@v3
    
    - name: Configure K8s
      uses: azure/k8s-set-context@v3
      with:
        kubeconfig: ${{ secrets.KUBECONFIG }}
    
    - name: Deploy
      id: deploy
      run: |
        if [ "${{ github.ref }}" == "refs/heads/main" ]; then
          helm upgrade --install my-app ./chart \
            --namespace staging \
            --values values-staging.yaml \
            --set image.tag=${{ github.sha }} \
            --atomic \
            --wait
        else
          helm upgrade --install my-app ./chart \
            --namespace production \
            --values values-prod.yaml \
            --set image.tag=${GITHUB_REF#refs/tags/} \
            --atomic \
            --wait
        fi
```

## License
This project is licensed under the MIT License.
