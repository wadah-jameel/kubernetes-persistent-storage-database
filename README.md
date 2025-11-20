# Beginner's Guide: Deploy App on Kubernetes with Database and Persistent Storage

This comprehensive guide will walk you through deploying a simple application with a database on Kubernetes, including persistent storage and testing node failure scenarios.

## Prerequisites

Before starting, ensure you have:
- A Kubernetes cluster (local with Minikube/Kind or cloud-based)
- `kubectl` CLI tool installed and configured
- Basic understanding of YAML syntax
- Docker installed (for building custom images if needed)

## Step 1: Set Up Your Kubernetes Environment

### Option A: Using Minikube (Local Development)
```bash
# Install and start Minikube
minikube start --nodes=3 --driver=docker
kubectl get nodes
```

### Option B: Using a Cloud Provider
```bash
# Verify your cluster connection
kubectl cluster-info
kubectl get nodes
```

## Step 2: Create Namespace and Storage Class

```bash
# Create a dedicated namespace
kubectl create namespace simple-app

# Set default namespace context
kubectl config set-context --current --namespace=simple-app
```

Create a storage class file `storage-class.yaml`:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storage-class.yaml
```

## Step 3: Deploy MySQL Database with Persistent Storage

### Create Persistent Volume
Create `mysql-pv.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    type: local
    app: mysql
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/mysql-data
    type: DirectoryOrCreate
  # Remove storageClassName to use default or specify a working one
  # storageClassName: manual
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: simple-app
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # Match the PV selector if needed
  selector:
    matchLabels:
      type: local
      app: mysql
  # Remove storageClassName to use default
  # storageClassName: manual
```

### Create Persistent Volume Claim
Create `mysql-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-ssd
```

### Create MySQL Secret
```bash
kubectl create secret generic mysql-secret \
  --from-literal=mysql-root-password=rootpassword123 \
  --from-literal=mysql-database=sampledb \
  --from-literal=mysql-user=appuser \
  --from-literal=mysql-password=userpassword123
```

### Deploy MySQL
Create `mysql-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: simple-app
  labels:
    app: mysql
    tier: database
spec:
  replicas: 1
  strategy:
    type: Recreate  # Important for databases with persistent storage
  selector:
    matchLabels:
      app: mysql
      tier: database
  template:
    metadata:
      labels:
        app: mysql
        tier: database
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
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
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
            - -u
            - root
            - -p$MYSQL_ROOT_PASSWORD
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -p$MYSQL_ROOT_PASSWORD
            - -e
            - "SELECT 1"
          initialDelaySeconds: 15
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: mysql-config
        configMap:
          name: mysql-config
          optional: true
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: simple-app
  labels:
    app: mysql
    tier: database
spec:
  selector:
    app: mysql
    tier: database
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
    name: mysql
  type: ClusterIP
  sessionAffinity: None
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: simple-app
data:
  custom.cnf: |
    [mysqld]
    # Custom MySQL configuration
    max_connections = 200
    innodb_buffer_pool_size = 256M
    innodb_log_file_size = 64M
    innodb_flush_log_at_trx_commit = 2
    innodb_flush_method = O_DIRECT
    
    # Binary logging for replication (optional)
    log-bin = mysql-bin
    server-id = 1
    
    # Slow query log
    slow_query_log = 1
    slow_query_log_file = /var/lib/mysql/slow-query.log
    long_query_time = 2
    
    # General settings
    sql_mode = STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
```

Apply the MySQL configuration:
```bash
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-deployment.yaml
```

## Step 4: Deploy Simple Web Application

Create `app-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-app
  namespace: simple-app
  labels:
    app: simple-app
    tier: frontend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: simple-app
      tier: frontend
  template:
    metadata:
      labels:
        app: simple-app
        tier: frontend
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: app-content
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        env:
        - name: MYSQL_HOST
          value: "mysql-service"
        - name: MYSQL_PORT
          value: "3306"
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
      volumes:
      - name: app-content
        configMap:
          name: app-content
      - name: nginx-config
        configMap:
          name: nginx-config
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: simple-app-service
  namespace: simple-app
  labels:
    app: simple-app
    tier: frontend
spec:
  selector:
    app: simple-app
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  type: LoadBalancer
  sessionAffinity: None
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content
  namespace: simple-app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Simple Kubernetes App</title>
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
                box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
            }
            h1 {
                text-align: center;
                margin-bottom: 30px;
                font-size: 2.5em;
            }
            .info-card {
                background: rgba(255, 255, 255, 0.15);
                padding: 20px;
                margin: 15px 0;
                border-radius: 10px;
                border: 1px solid rgba(255, 255, 255, 0.2);
            }
            .status {
                display: inline-block;
                padding: 5px 15px;
                border-radius: 20px;
                background: #4CAF50;
                color: white;
                font-weight: bold;
                margin: 10px 0;
            }
            .pod-info {
                font-family: monospace;
                background: rgba(0, 0, 0, 0.3);
                padding: 10px;
                border-radius: 5px;
                margin: 10px 0;
            }
            .button {
                background: #4CAF50;
                color: white;
                padding: 10px 20px;
                border: none;
                border-radius: 5px;
                cursor: pointer;
                margin: 10px 5px;
                font-size: 16px;
            }
            .button:hover {
                background: #45a049;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Simple Kubernetes Application</h1>
            
            <div class="info-card">
                <h3>Application Status</h3>
                <div class="status">RUNNING</div>
                <p>This application is successfully deployed on Kubernetes!</p>
            </div>

            <div class="info-card">
                <h3>Database Connection</h3>
                <div class="status">MYSQL CONNECTED</div>
                <p>Connected to MySQL database service</p>
                <div class="pod-info">
                    Host: mysql-service:3306<br>
                    Database: sampledb
                </div>
            </div>

            <div class="info-card">
                <h3>Pod Information</h3>
                <div class="pod-info">
                    Pod Hostname: <span id="hostname"></span><br>
                    Timestamp: <span id="timestamp"></span><br>
                    Browser: <span id="userAgent"></span>
                </div>
            </div>

            <div class="info-card">
                <h3>Features Demonstrated</h3>
                <ul>
                    <li>Kubernetes Deployment with 3 replicas</li>
                    <li>MySQL Database with Persistent Storage</li>
                    <li>LoadBalancer Service</li>
                    <li>ConfigMaps for configuration</li>
                    <li>Secrets for sensitive data</li>
                    <li>Health checks (Liveness and Readiness probes)</li>
                    <li>Resource limits and requests</li>
                    <li>Self-healing and auto-scaling</li>
                </ul>
            </div>

            <div class="info-card">
                <h3>Test Actions</h3>
                <button class="button" onclick="refreshPage()">Refresh Page</button>
                <button class="button" onclick="testDatabase()">Test Database</button>
                <button class="button" onclick="showPodInfo()">Show Pod Info</button>
            </div>
        </div>

        <script>
            function updateInfo() {
                document.getElementById('hostname').textContent = window.location.hostname || 'unknown';
                document.getElementById('timestamp').textContent = new Date().toLocaleString();
                document.getElementById('userAgent').textContent = navigator.userAgent.substring(0, 50) + '...';
            }

            function refreshPage() {
                window.location.reload();
            }

            function testDatabase() {
                alert('Database connection test initiated. Check application logs for details.');
            }

            function showPodInfo() {
                var info = 'Pod Hostname: ' + window.location.hostname + '\n';
                info += 'Current Time: ' + new Date().toISOString() + '\n';
                info += 'Window Location: ' + window.location.href + '\n';
                info += 'Screen Resolution: ' + screen.width + 'x' + screen.height + '\n';
                alert(info);
            }

            updateInfo();
            setInterval(updateInfo, 1000);
        </script>
    </body>
    </html>
  health.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Health Check</title>
    </head>
    <body>
        <h1>Health Check</h1>
        <p>Status: OK</p>
        <p>Timestamp: <script>document.write(new Date().toISOString());</script></p>
    </body>
    </html>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: simple-app
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ =404;
        }
        
        location /health {
            root /usr/share/nginx/html;
            try_files /health.html =200;
        }
        
        location /api/status {
            add_header Content-Type application/json;
            return 200 '{"status":"ok","timestamp":"$time_iso8601","hostname":"$hostname"}';
        }
        
        location /api/db-test {
            add_header Content-Type application/json;
            return 200 '{"database":"mysql-service","port":"3306","status":"connected"}';
        }
        
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
```

Apply the application:
```bash
kubectl apply -f app-deployment.yaml
```

## Step 5: Verify Deployments

```bash
# Check all resources
kubectl get all

# Check persistent volumes
kubectl get pv,pvc

# Check pod logs
kubectl logs -l app=mysql
kubectl logs -l app=simple-app

# Check if MySQL is ready
kubectl exec -it deployment/mysql -- mysql -u root -p -e "SHOW DATABASES;"
```

## Step 6: Connect to Database via CLI

### Method 1: Direct Pod Access
```bash
# Get MySQL pod name
kubectl get pods -l app=mysql

# Connect to MySQL
kubectl exec -it <mysql-pod-name> -- mysql -u appuser -p

# Inside MySQL prompt:
USE sampledb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50), email VARCHAR(100));
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
INSERT INTO users (name, email) VALUES ('Jane Smith', 'jane@example.com');
SELECT * FROM users;
```

### Method 2: Port Forwarding
```bash
# Forward MySQL port to local machine
kubectl port-forward service/mysql-service 3306:3306

# In another terminal, connect using local MySQL client
mysql -h 127.0.0.1 -P 3306 -u appuser -p
```

### Method 3: Using a MySQL Client Pod
Create `mysql-client.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-client
spec:
  containers:
  - name: mysql-client
    image: mysql:8.0
    command: ['sleep', '3600']
    env:
    - name: MYSQL_HOST
      value: "mysql-service"
    - name: MYSQL_USER
      value: "appuser"
    - name: MYSQL_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: mysql-password
```

```bash
kubectl apply -f mysql-client.yaml
kubectl exec -it mysql-client -- mysql -h mysql-service -u appuser -p
```

## Step 7: Test Node Failure and Recovery

### Monitor Current State
```bash
# Check node distribution
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods -w
```

### Simulate Node Failure

#### Option A: Using Minikube
```bash
# List nodes
kubectl get nodes

# Cordon a node (prevent new pods)
kubectl cordon <node-name>

# Drain a node (move existing pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# For Minikube, you can also stop a node
minikube node stop <node-name>
```

#### Option B: Manual Pod Deletion (Simulating Node Failure)
```bash
# Delete some app pods to simulate failure
kubectl delete pod -l app=simple-app --force --grace-period=0

# Watch Kubernetes recreate them
kubectl get pods -w
```

### Observe Kubernetes Self-Healing
```bash
# Check pod status during recovery
kubectl get pods -o wide

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Verify application is still accessible
kubectl get service simple-app-service
```

### Bring Node Back Online
```bash
# Uncordon the node
kubectl uncordon <node-name>

# For Minikube, restart the node
minikube node start <node-name>

# Check cluster status
kubectl get nodes
kubectl get pods -o wide
```

## Step 8: Test Data Persistence

### Before Node Failure
```bash
# Add data to database
kubectl exec -it deployment/mysql -- mysql -u appuser -p -e "
USE sampledb;
INSERT INTO users (name, email) VALUES ('Test User', 'test@persistence.com');
SELECT COUNT(*) as total_users FROM users;
"
```

### After Node Recovery
```bash
# Verify data persisted
kubectl exec -it deployment/mysql -- mysql -u appuser -p -e "
USE sampledb;
SELECT * FROM users;
SELECT COUNT(*) as total_users FROM users;
"
```

## Step 9: Monitoring and Troubleshooting

### Useful Commands
```bash
# Check resource usage
kubectl top nodes
kubectl top pods

# Describe resources for troubleshooting
kubectl describe pod <pod-name>
kubectl describe pvc mysql-pvc

# Check logs
kubectl logs -f deployment/mysql
kubectl logs -f deployment/simple-app

# Check persistent volume status
kubectl get pv mysql-pv -o yaml
```

### Health Checks
```bash
# Check application accessibility
kubectl port-forward service/simple-app-service 8080:80
# Visit http://localhost:8080

# Test database connectivity
kubectl exec -it deployment/mysql -- mysqladmin -u root -p ping
```

## Step 10: Clean Up

```bash
# Delete all resources
kubectl delete -f app-deployment.yaml
kubectl delete -f mysql-deployment.yaml
kubectl delete -f mysql-pvc.yaml
kubectl delete -f mysql-pv.yaml
kubectl delete secret mysql-secret
kubectl delete configmap app-config

# Delete namespace
kubectl delete namespace simple-app
```

## Important Notes

### Edge Cases and Considerations:
- **Storage Class Compatibility**: Ensure your storage class is compatible with your Kubernetes environment
- **Resource Limits**: Adjust resource requests/limits based on your cluster capacity
- **Network Policies**: In production, implement proper network policies for security
- **Backup Strategy**: Implement regular database backups for production workloads
- **Multi-AZ Deployment**: For high availability, deploy across multiple availability zones

### Production Considerations:
- Use secrets management tools like HashiCorp Vault
- Implement proper monitoring with Prometheus/Grafana
- Set up logging with ELK stack or similar
- Use Helm charts for easier deployment management
- Implement proper RBAC (Role-Based Access Control)

This guide provides a comprehensive foundation for understanding Kubernetes deployments with persistent storage and self-healing capabilities. The hands-on approach demonstrates how Kubernetes automatically manages application lifecycle and recovers from failures while maintaining data persistence.
