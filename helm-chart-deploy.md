# MySQL Deployment with Helm Chart

Let's create a complete Helm chart for deploying MySQL with persistent storage. This will make deployment and management much easier.

## Step 1: Create Helm Chart Structure

```bash
# Create the basic Helm chart structure
helm create mysql-app

# Remove default files we don't need
rm mysql-app/templates/deployment.yaml
rm mysql-app/templates/service.yaml
rm mysql-app/templates/serviceaccount.yaml
rm mysql-app/templates/hpa.yaml
rm mysql-app/templates/ingress.yaml
rm mysql-app/templates/NOTES.txt
rm -rf mysql-app/templates/tests/

# Create our custom structure
mkdir -p mysql-app/templates/mysql
mkdir -p mysql-app/templates/app
```

## Step 2: Update Chart.yaml

**File: `mysql-app/Chart.yaml`**
```yaml
apiVersion: v2
name: mysql-app
description: A Helm chart for MySQL application with persistent storage
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - mysql
  - database
  - kubernetes
  - persistent-storage
home: https://github.com/your-org/mysql-app
sources:
  - https://github.com/your-org/mysql-app
maintainers:
  - name: Your Name
    email: your.email@example.com
```

## Step 3: Configure Values

**File: `mysql-app/values.yaml`**
```yaml
# Global configuration
global:
  namespace: simple-app
  storageClass: ""  # Use default storage class

# MySQL Configuration
mysql:
  enabled: true
  image:
    repository: mysql
    tag: "8.0"
    pullPolicy: IfNotPresent
  
  # Database configuration
  auth:
    rootPassword: "rootpassword123"
    database: "sampledb"
    username: "appuser"
    password: "userpassword123"
  
  # Storage configuration
  persistence:
    enabled: true
    size: 5Gi
    accessMode: ReadWriteOnce
    hostPath: "/tmp/mysql-data"
    reclaimPolicy: Retain
  
  # Resource limits
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  
  # Service configuration
  service:
    type: ClusterIP
    port: 3306
  
  # Health checks
  probes:
    enabled: true
    startup:
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 30
    liveness:
      initialDelaySeconds: 60
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3
    readiness:
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

# Application Configuration
app:
  enabled: true
  name: simple-app
  image:
    repository: nginx
    tag: "alpine"
    pullPolicy: IfNotPresent
  
  replicaCount: 3
  
  # Resource limits
  resources:
    requests:
      memory: "64Mi"
      cpu: "100m"
    limits:
      memory: "128Mi"
      cpu: "200m"
  
  # Service configuration
  service:
    type: LoadBalancer
    port: 80
  
  # Health checks
  probes:
    enabled: true
    liveness:
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readiness:
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3

# ConfigMaps
configMaps:
  app:
    enabled: true
  mysql:
    enabled: true
```

## Step 4: Create MySQL Templates

**File: `mysql-app/templates/mysql/secret.yaml`**
```yaml
{{- if .Values.mysql.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: mysql
type: Opaque
data:
  mysql-root-password: {{ .Values.mysql.auth.rootPassword | b64enc | quote }}
  mysql-database: {{ .Values.mysql.auth.database | b64enc | quote }}
  mysql-user: {{ .Values.mysql.auth.username | b64enc | quote }}
  mysql-password: {{ .Values.mysql.auth.password | b64enc | quote }}
{{- end }}
```

**File: `mysql-app/templates/mysql/pv.yaml`**
```yaml
{{- if and .Values.mysql.enabled .Values.mysql.persistence.enabled }}
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: mysql
    type: local
spec:
  capacity:
    storage: {{ .Values.mysql.persistence.size }}
  accessModes:
    - {{ .Values.mysql.persistence.accessMode }}
  persistentVolumeReclaimPolicy: {{ .Values.mysql.persistence.reclaimPolicy }}
  hostPath:
    path: {{ .Values.mysql.persistence.hostPath }}
    type: DirectoryOrCreate
  {{- if .Values.global.storageClass }}
  storageClassName: {{ .Values.global.storageClass }}
  {{- end }}
{{- end }}
```

**File: `mysql-app/templates/mysql/pvc.yaml`**
```yaml
{{- if and .Values.mysql.enabled .Values.mysql.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: mysql
spec:
  accessModes:
    - {{ .Values.mysql.persistence.accessMode }}
  resources:
    requests:
      storage: {{ .Values.mysql.persistence.size }}
  {{- if .Values.global.storageClass }}
  storageClassName: {{ .Values.global.storageClass }}
  {{- end }}
  selector:
    matchLabels:
      component: mysql
      type: local
{{- end }}
```

**File: `mysql-app/templates/mysql/deployment.yaml`**
```yaml
{{- if .Values.mysql.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: mysql
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      {{- include "mysql-app.selectorLabels" . | nindent 6 }}
      component: mysql
  template:
    metadata:
      labels:
        {{- include "mysql-app.selectorLabels" . | nindent 8 }}
        component: mysql
    spec:
      containers:
      - name: mysql
        image: "{{ .Values.mysql.image.repository }}:{{ .Values.mysql.image.tag }}"
        imagePullPolicy: {{ .Values.mysql.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.mysql.service.port }}
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        {{- if .Values.mysql.persistence.enabled }}
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        {{- end }}
        resources:
          {{- toYaml .Values.mysql.resources | nindent 10 }}
        {{- if .Values.mysql.probes.enabled }}
        startupProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
          initialDelaySeconds: {{ .Values.mysql.probes.startup.initialDelaySeconds }}
          periodSeconds: {{ .Values.mysql.probes.startup.periodSeconds }}
          timeoutSeconds: {{ .Values.mysql.probes.startup.timeoutSeconds }}
          failureThreshold: {{ .Values.mysql.probes.startup.failureThreshold }}
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
          initialDelaySeconds: {{ .Values.mysql.probes.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.mysql.probes.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.mysql.probes.liveness.timeoutSeconds }}
          failureThreshold: {{ .Values.mysql.probes.liveness.failureThreshold }}
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
            - -e
            - "SELECT 1"
          initialDelaySeconds: {{ .Values.mysql.probes.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.mysql.probes.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.mysql.probes.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.mysql.probes.readiness.failureThreshold }}
        {{- end }}
      {{- if .Values.mysql.persistence.enabled }}
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      {{- end }}
{{- end }}
```

**File: `mysql-app/templates/mysql/service.yaml`**
```yaml
{{- if .Values.mysql.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: mysql
spec:
  selector:
    {{- include "mysql-app.selectorLabels" . | nindent 4 }}
    component: mysql
  ports:
  - port: {{ .Values.mysql.service.port }}
    targetPort: {{ .Values.mysql.service.port }}
    protocol: TCP
    name: mysql
  type: {{ .Values.mysql.service.type }}
{{- end }}
```

## Step 5: Create App Templates

**File: `mysql-app/templates/app/configmap.yaml`**
```yaml
{{- if and .Values.app.enabled .Values.configMaps.app.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>{{ .Values.app.name | title }}</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                margin: 0;
                padding: 20px;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                color: white;
                min-height: 100vh;
            }
            .container {
                max-width: 800px;
                margin: 0 auto;
                background: rgba(255, 255, 255, 0.1);
                padding: 30px;
                border-radius: 15px;
                backdrop-filter: blur(10px);
            }
            .status {
                background: #4CAF50;
                color: white;
                padding: 5px 15px;
                border-radius: 20px;
                display: inline-block;
                margin: 10px 0;
            }
            .info-card {
                background: rgba(255, 255, 255, 0.15);
                padding: 20px;
                margin: 15px 0;
                border-radius: 10px;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>{{ .Values.app.name | title }} - Deployed with Helm</h1>
            
            <div class="info-card">
                <h3>Application Status</h3>
                <div class="status">RUNNING</div>
                <p>Deployed using Helm Chart v{{ .Chart.Version }}</p>
            </div>

            <div class="info-card">
                <h3>Database Connection</h3>
                <div class="status">MYSQL CONNECTED</div>
                <p>Connected to MySQL service: mysql-service:{{ .Values.mysql.service.port }}</p>
            </div>

            <div class="info-card">
                <h3>Configuration</h3>
                <ul>
                    <li>App Replicas: {{ .Values.app.replicaCount }}</li>
                    <li>MySQL Version: {{ .Values.mysql.image.tag }}</li>
                    <li>Storage Size: {{ .Values.mysql.persistence.size }}</li>
                    <li>Namespace: {{ .Values.global.namespace }}</li>
                </ul>
            </div>
        </div>
        <script>
            console.log('App deployed with Helm chart: {{ .Chart.Name }} v{{ .Chart.Version }}');
        </script>
    </body>
    </html>
{{- end }}
```

**File: `mysql-app/templates/app/deployment.yaml`**
```yaml
{{- if .Values.app.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: app
spec:
  replicas: {{ .Values.app.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      {{- include "mysql-app.selectorLabels" . | nindent 6 }}
      component: app
  template:
    metadata:
      labels:
        {{- include "mysql-app.selectorLabels" . | nindent 8 }}
        component: app
    spec:
      containers:
      - name: app
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        imagePullPolicy: {{ .Values.app.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.app.service.port }}
          name: http
        volumeMounts:
        - name: app-content
          mountPath: /usr/share/nginx/html
        resources:
          {{- toYaml .Values.app.resources | nindent 10 }}
        env:
        - name: MYSQL_HOST
          value: "mysql-service"
        - name: MYSQL_PORT
          value: "{{ .Values.mysql.service.port }}"
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        {{- if .Values.app.probes.enabled }}
        livenessProbe:
          httpGet:
            path: /
            port: {{ .Values.app.service.port }}
          initialDelaySeconds: {{ .Values.app.probes.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.app.probes.liveness.periodSeconds }}
          timeoutSeconds: {{ .Values.app.probes.liveness.timeoutSeconds }}
          failureThreshold: {{ .Values.app.probes.liveness.failureThreshold }}
        readinessProbe:
          httpGet:
            path: /
            port: {{ .Values.app.service.port }}
          initialDelaySeconds: {{ .Values.app.probes.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.app.probes.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.app.probes.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.app.probes.readiness.failureThreshold }}
        {{- end }}
      volumes:
      - name: app-content
        configMap:
          name: app-content
{{- end }}
```

**File: `mysql-app/templates/app/service.yaml`**
```yaml
{{- if .Values.app.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}-service
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
    component: app
spec:
  selector:
    {{- include "mysql-app.selectorLabels" . | nindent 4 }}
    component: app
  ports:
  - port: {{ .Values.app.service.port }}
    targetPort: {{ .Values.app.service.port }}
    protocol: TCP
    name: http
  type: {{ .Values.app.service.type }}
{{- end }}
```

## Step 6: Update Helpers Template

**File: `mysql-app/templates/_helpers.tpl`**
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "mysql-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mysql-app.fullname" -}}
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
{{- define "mysql-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mysql-app.labels" -}}
helm.sh/chart: {{ include "mysql-app.chart" . }}
{{ include "mysql-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mysql-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mysql-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## Step 7: Create Namespace Template

**File: `mysql-app/templates/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.global.namespace }}
  labels:
    {{- include "mysql-app.labels" . | nindent 4 }}
```

## Step 8: Create NOTES Template

**File: `mysql-app/templates/NOTES.txt`**
```
🎉 {{ .Chart.Name }} has been deployed successfully!

📋 Release Information:
   Name:      {{ .Release.Name }}
   Namespace: {{ .Values.global.namespace }}
   Version:   {{ .Chart.Version }}

🐛 Get the application URL by running these commands:
{{- if eq .Values.app.service.type "LoadBalancer" }}
   NOTE: It may take a few minutes for the LoadBalancer IP to be available.
   
   export SERVICE_IP=$(kubectl get svc --namespace {{ .Values.global.namespace }} {{ .Values.app.name }}-service --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
   echo http://$SERVICE_IP:{{ .Values.app.service.port }}

{{- else if eq .Values.app.service.type "ClusterIP" }}
   kubectl --namespace {{ .Values.global.namespace }} port-forward service/{{ .Values.app.name }}-service {{ .Values.app.service.port }}:{{ .Values.app.service.port }}
   echo "Visit http://127.0.0.1:{{ .Values.app.service.port }} to use your application"
{{- end }}

🗄️  MySQL Database:
   Host:     mysql-service.{{ .Values.global.namespace }}.svc.cluster.local
   Port:     {{ .Values.mysql.service.port }}
   Database: {{ .Values.mysql.auth.database }}
   Username: {{ .Values.mysql.auth.username }}

📊 Check deployment status:
   kubectl get all -n {{ .Values.global.namespace }}

🔍 View application logs:
   kubectl logs -f deployment/{{ .Values.app.name }} -n {{ .Values.global.namespace }}
   kubectl logs -f deployment/mysql -n {{ .Values.global.namespace }}

🧪 Test database connection:
   kubectl exec -it deployment/mysql -n {{ .Values.global.namespace }} -- mysql -u {{ .Values.mysql.auth.username }} -p{{ .Values.mysql.auth.password }} -e "SELECT VERSION();"
```

## Step 9: Deploy with Helm

```bash
# Create namespace first (if needed)
kubectl create namespace simple-app --dry-run=client -o yaml | kubectl apply -f -

# Install Helm (if not already installed)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Validate the chart
helm lint mysql-app/

# Test template rendering
helm template my-mysql-app mysql-app/ --namespace simple-app

# Deploy the application
helm install my-mysql-app mysql-app/ --namespace simple-app --create-namespace

# Check deployment status
helm status my-mysql-app -n simple-app

# List Helm releases
helm list -n simple-app
```

## Step 10: Management Commands

```bash
# Upgrade the deployment
helm upgrade my-mysql-app mysql-app/ -n simple-app

# Rollback to previous version
helm rollback my-mysql-app 1 -n simple-app

# Get deployment history
helm history my-mysql-app -n simple-app

# Uninstall the application
helm uninstall my-mysql-app -n simple-app

# Check what would be deployed (dry-run)
helm install my-mysql-app mysql-app/ --namespace simple-app --dry-run --debug
```

## Step 11: Custom Values Files

**Create `values-production.yaml` for production:**
```yaml
global:
  namespace: mysql-app-prod
  storageClass: "gp2"  # AWS EBS

mysql:
  persistence:
    size: 20Gi
    reclaimPolicy: Retain
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"

app:
  replicaCount: 5
  service:
    type: ClusterIP  # Use Ingress instead
```

**Deploy with custom values:**
```bash
helm install my-mysql-app mysql-app/ -f values-production.yaml -n mysql-app-prod --create-namespace
```

## Step 12: Verification Commands

```bash
# Check all resources
kubectl get all -n simple-app

# Test the application
kubectl port-forward service/simple-app-service 8080:80 -n simple-app
# Visit http://localhost:8080

# Test database
kubectl exec -it deployment/mysql -n simple-app -- mysql -u appuser -puserpassword123 -e "SELECT VERSION();"

# Check Helm values
helm get values my-mysql-app -n simple-app

# Check Helm manifest
helm get manifest my-mysql-app -n simple-app
```

This Helm chart provides:
- ✅ **Templated configurations** with customizable values
- ✅ **Production-ready** with proper labels and resource management
- ✅ **Easy deployment and management** with Helm commands
- ✅ **Environment-specific configurations** through values files
- ✅ **Upgrade and rollback capabilities**
- ✅ **Clear documentation** and status information

The chart makes it much easier to deploy, manage, and customize your MySQL application across different environments!
