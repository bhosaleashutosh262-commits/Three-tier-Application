# Architecture

```text
Frontend (Deployment + Service)
          |
          v
Backend (Deployment + Service)
          |
          v
MySQL (Deployment + Service)
          |
          v
EFS PV/PVC
```

---

# 1. backend-deployment.yaml

Replace `<BACKEND_IMAGE>` with your backend Docker image.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: student-app
  labels:
    app: backend

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
        image: <BACKEND_IMAGE>

        ports:
        - containerPort: 8080

        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql-service:3306/student_db

        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER

        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
```

---

# 2. backend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: student-app

spec:
  selector:
    app: backend

  ports:
  - port: 80
    targetPort: 8080

  type: ClusterIP
```

---

# 3. frontend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: student-app

spec:
  replicas: 3

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
        image: <FRONTEND_IMAGE>

        ports:
        - containerPort: 80
```

---

# 4. frontend-service.yaml

Use LoadBalancer to access the application from the internet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: student-app

spec:
  selector:
    app: frontend

  ports:
  - port: 80
    targetPort: 80

  type: LoadBalancer
```

---

# 5. mysql-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: student-app

type: Opaque

stringData:
  MYSQL_ROOT_PASSWORD: redhat123
  MYSQL_DATABASE: student_db
  MYSQL_USER: admin
  MYSQL_PASSWORD: redhat123
```

---

# 6. mysql-init-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-init-config
  namespace: student-app

data:
  init.sql: |
    USE student_db;

    CREATE TABLE IF NOT EXISTS students (
      id BIGINT(20) NOT NULL AUTO_INCREMENT,
      name VARCHAR(255),
      email VARCHAR(255),
      course VARCHAR(255),
      student_class VARCHAR(255),
      percentage DOUBLE,
      branch VARCHAR(255),
      mobile_number VARCHAR(255),
      PRIMARY KEY (id)
    );
```

---

# 7. efs-pv-pvc.yaml

Replace with your actual EFS ID.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv

spec:
  capacity:
    storage: 5Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteMany

  persistentVolumeReclaimPolicy: Retain

  storageClassName: efs-sc

  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0abc123456789 <--- update this 

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
  namespace: student-app

spec:
  accessModes:
    - ReadWriteMany

  storageClassName: efs-sc

  resources:
    requests:
      storage: 5Gi
```

---

# 8. mysql-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: student-app

spec:
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
        image: mysql:8.0

        ports:
        - containerPort: 3306

        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_ROOT_PASSWORD

        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_DATABASE

        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER

        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD

        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql

        - name: init-script
          mountPath: /docker-entrypoint-initdb.d

      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: efs-pvc

      - name: init-script
        configMap:
          name: mysql-init-config
```

---

# 9. mysql-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: student-app

spec:
  selector:
    app: mysql

  ports:
  - port: 3306
    targetPort: 3306

  type: ClusterIP
```

---

# Deployment Commands

```bash
kubectl create namespace student-app

kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-init-configmap.yaml

kubectl apply -f efs-pv-pvc.yaml

kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml

kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml

kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

---

# Verification

```bash
kubectl get pods -n student-app

kubectl get svc -n student-app

kubectl get pvc -n student-app

kubectl get pv
```

Get frontend URL:

```bash
kubectl get svc frontend-service -n student-app
```

Wait for the `EXTERNAL-IP` and open it in your browser.
