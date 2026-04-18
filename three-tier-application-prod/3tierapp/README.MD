3-TIER APPLICATION 

Replace Kubernetes Secrets → AWS Secrets Manager
Use Stateful DB (MySQL StatefulSet)
Add PersistentVolume (PV) + PersistentVolumeClaim (PVC)
Full 3-tier architecture (frontend + backend + DB)
Include Namespace, Services, Ingress

Below is a clean, end-to-end implementation with explanation.

🔷 1. Architecture (Final)

Frontend (Nginx) → Backend (Node.js) → Database (MySQL StatefulSet)

Secrets flow:

Stored in 👉 AWS Secrets Manager
Synced to K8s via 👉 External Secrets Operator
Used as env vars inside pods

Storage:

MySQL uses PVC → PV (persistent disk)
🔷 2. Step-by-Step Implementation
✅ Step 1: Create Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: three-tier
✅ Step 2: Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
✅ Step 3: Store Secret in AWS

Create secret:

Name: prod/mysql-secret

Value:

{
  "username": "admin",
  "password": "mypassword123"
}
✅ Step 4: SecretStore
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secret-store
  namespace: three-tier
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
✅ Step 5: ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mysql-secret
  namespace: three-tier
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore
  target:
    name: mysql-secret
  data:
  - secretKey: MYSQL_USER
    remoteRef:
      key: prod/mysql-secret
      property: username
  - secretKey: MYSQL_PASSWORD
    remoteRef:
      key: prod/mysql-secret
      property: password
🔷 3. DATABASE TIER (StatefulSet + PV + PVC)
✅ Step 6: Persistent Volume (PV)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql
✅ Step 7: Persistent Volume Claim (PVC)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: three-tier
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
✅ Step 8: MySQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: three-tier
spec:
  serviceName: mysql-headless
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
✅ Step 9: Headless Service (Required for StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: three-tier
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
🔷 4. BACKEND TIER
✅ Step 10: Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: node:18
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: mysql-headless
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
✅ Step 11: Backend Service
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: three-tier
spec:
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
🔷 5. FRONTEND TIER
✅ Step 12: Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
✅ Step 13: Frontend Service
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: three-tier
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
🔷 6. INGRESS
✅ Step 14: Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: three-tier-ingress
  namespace: three-tier
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
🔷 7. Deployment Order
kubectl apply -f namespace.yaml
kubectl apply -f secretstore.yaml
kubectl apply -f externalsecret.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f mysql.yaml
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl apply -f ingress.yaml
🔷 8. How Everything Works (Simple Flow)
Secrets stored in 👉 AWS Secrets Manager
Synced to Kubernetes via 👉 External Secrets Operator
MySQL StatefulSet uses PVC → data persists
Backend fetches DB credentials securely
Frontend exposed via Ingress
🔷 9. Important Production Notes
❌ Avoid hostPath → use EBS (in EKS)
✅ Use StorageClass instead of static PV
✅ Enable TLS in Ingress (ALB Ingress Controller)
✅ Use RDS instead of MySQL pod (best practice)
✅ Add resource limits + probes