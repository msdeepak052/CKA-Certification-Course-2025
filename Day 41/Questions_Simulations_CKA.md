```bash
# Clean if anything exists
kubectl delete ns mariadb --ignore-not-found=true
sleep 2

# Create namespace
kubectl create ns mariadb

# Create local storage directory
mkdir -p /mnt/mariadb-data

# ----------------------------
# STEP 1: Create Retained PV
# ----------------------------
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity:
    storage: 250Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/mariadb-data
EOF

# ----------------------------
# STEP 2: Create TEMP PVC (will be deleted later)
# ----------------------------
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: temp-mariadb-pvc
  namespace: mariadb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
EOF

# ----------------------------
# STEP 3: Create MariaDB Deployment (WORKING)
# ----------------------------
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.5
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpass
        volumeMounts:
        - name: mariadb-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-storage
        persistentVolumeClaim:
          claimName: temp-mariadb-pvc
EOF

# Wait for pod
kubectl wait --for=condition=available deployment/mariadb -n mariadb --timeout=60s

# ----------------------------
# STEP 4: Insert Test Data
# ----------------------------
POD=$(kubectl get pod -n mariadb -l app=mariadb -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n mariadb $POD -- mysql -uroot -prootpass -e "CREATE DATABASE cka_test;"
kubectl exec -n mariadb $POD -- mysql -uroot -prootpass -e "SHOW DATABASES;"

# ----------------------------
# STEP 5: Simulate ACCIDENT
# ----------------------------
kubectl delete deployment mariadb -n mariadb
kubectl delete pvc temp-mariadb-pvc -n mariadb

# PV now becomes Released (Retained)
kubectl patch pv mariadb-pv -p '{"spec":{"claimRef": null}}'

# ----------------------------
# STEP 6: Provide Broken Deployment File
# ----------------------------
cat <<EOF > /mariadb-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.5
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpass
        volumeMounts:
        - name: mariadb-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-storage
        emptyDir: {}
EOF

echo "------------------------------------------------"
echo "🔥 SCENARIO READY 🔥"
echo "------------------------------------------------"
echo "Namespace: mariadb"
echo "One PV exists (Retain policy)"
echo "Deployment deleted"
echo "PVC deleted"
echo "Broken deployment file at /mariadb-deploy.yaml"
echo ""
echo "YOUR TASK:"
echo "1. Create PVC named 'mariadb' in mariadb namespace (RWO, 250Mi)"
echo "2. Edit /mariadb-deploy.yaml to use that PVC"
echo "3. Apply it"
echo "4. Ensure deployment is running and data preserved"
echo "------------------------------------------------"
```
