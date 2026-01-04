# 🚀 Deploying WordPress on Kubernetes (simple setup)

> **⚠️ Disclaimer:** This guide is designed for educational and lab environments. It uses **NodePort** for external access and **Local Path Provisioner** for storage, which are suitable for testing but require additional security hardening for production use.

## 📋 Prerequisites

Before proceeding, ensure you have:

1. A running Kubernetes Cluster (v1.20+).
2. `kubectl` configured to access the cluster.
3. A **Default StorageClass** installed (e.g., Rancher Local Path Provisioner) to handle Persistent Volume Claims (PVCs).

## 🛠 Architecture Overview

We will deploy a classic 3-tier application in a dedicated namespace named `wordpress`:

| Component      | Resource Type | Description                                                    |
| -------------- | ------------- | -------------------------------------------------------------- |
| **Namespace**  | Namespace     | Isolates resources from the rest of the cluster.               |
| **Storage**    | PVC           | Two 5Gi volumes for MySQL data and WordPress files.            |
| **Database**   | Deployment    | MySQL 5.7 with a dedicated user and database.                  |
| **Frontend**   | Deployment    | Official WordPress image.                                      |
| **Networking** | Service       | ClusterIP for internal DB comms, NodePort for external access. |

***

## 📄 The Manifest File (`wordpress.yaml`)

Save the following code into a file named `wordpress.yaml`. This is a self-contained manifest that sets up everything.

````yaml
# 🚀 Deploying WordPress on Kubernetes (From Scratch)

> **⚠️ Disclaimer:** This guide is designed for educational and lab environments. It uses **NodePort** for external access and **Local Path Provisioner** for storage, which are suitable for testing but require additional security hardening for production use.

## 📋 Prerequisites

Before proceeding, ensure you have:
1.  A running Kubernetes Cluster (v1.20+).
2.  `kubectl` configured to access the cluster.
3.  A **Default StorageClass** installed (e.g., Rancher Local Path Provisioner) to handle Persistent Volume Claims (PVCs).

## 🛠 Architecture Overview

We will deploy a classic 3-tier application in a dedicated namespace named `wordpress`:

| Component | Resource Type | Description |
| :--- | :--- | :--- |
| **Namespace** | Namespace | Isolates resources from the rest of the cluster. |
| **Storage** | PVC | Two 5Gi volumes for MySQL data and WordPress files. |
| **Database** | Deployment | MySQL 5.7 with a dedicated user and database. |
| **Frontend** | Deployment | Official WordPress image. |
| **Networking** | Service | ClusterIP for internal DB comms, NodePort for external access. |

---

## 📄 The Manifest File (`wordpress.yaml`)

Save the following code into a file named `wordpress.yaml`. This is a self-contained manifest that sets up everything.

```yaml
# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
# 2. Secret (Database Credentials)
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
  namespace: wordpress
type: Opaque
stringData:
  # This password is used for both root and the wordpress user
  password: "MySecretPassword123"
---
# 3. PVC for Database
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# 4. PVC for WordPress Files
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# 5. Service for Database (Internal)
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
# 6. Deployment for Database (MySQL 5.7)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        # Define Root Password
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        # Create a specific Database & User (Best Practice)
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
# 7. Service for WordPress (External Access)
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
  namespace: wordpress
spec:
  ports:
    - port: 80
      nodePort: 30080 # Exposed Port on Host
  selector:
    app: wordpress
    tier: frontend
  type: NodePort
---
# 8. Deployment for WordPress Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        # Use FQDN to prevent DNS resolution issues
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql.wordpress.svc.cluster.local
        # Connect with the dedicated user created above
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
````

***

## ⚡ Deployment Instructions

### Step 1: Clean Up (If retrying)

If you have deployed this before and want a fresh start (essential for Database initialization), delete the old resources and PVCs:

```shellscript
kubectl delete -f wordpress.yaml
kubectl delete pvc -n wordpress --all
```

### Step 2: Apply the Configuration

Run the following command to create all resources:

```shellscript
kubectl apply -f wordpress.yaml
```

### Step 3: Verification

Check the status of the pods. It may take 1-3 minutes for the images to pull and start.

```shellscript
kubectl get pods -n wordpress
```

**Expected Output:**

```tex
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-7b969...                 1/1     Running   0          2m
wordpress-mysql-7d6...             1/1     Running   0          2m
```

***

## 🌍 Access the Website

You can access the WordPress installation page using the **IP address of any Worker Node** and port **30080**.

**URL:**`http://<WORKER-NODE-IP>:30080`

Example:

* `http://192.168.100.21:30080`
* `http://192.168.100.22:30080`

***

## 🐞 Troubleshooting

If you see an "Error establishing a database connection":

1. **Check Logs:**
   ```shellscript
   kubectl logs -n wordpress -l app=wordpress
   ```
2. **Verify Database Initialization:** Often, if the PVC wasn't cleaned up from a previous failed attempt, MySQL won't create the new user.If you see "Access denied", perform **Step 1 (Clean Up)** again to force a fresh database creation.
   ```shellscript
   kubectl logs -n wordpress -l app=wordpress --tail=20
   ```

