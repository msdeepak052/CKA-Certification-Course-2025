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

```bash

echo "--------------------------------------"
echo "🔍 CKA SOLUTION VALIDATION"
echo "--------------------------------------"

PASS=true

# PVC exists
if kubectl get pvc mariadb -n mariadb &>/dev/null; then
  echo "✅ PVC exists"
else
  echo "❌ PVC 'mariadb' not found"
  PASS=false
fi

# AccessMode
MODE=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.spec.accessModes[0]}' 2>/dev/null)
if [ "$MODE" = "ReadWriteOnce" ]; then
  echo "✅ AccessMode correct"
else
  echo "❌ AccessMode is not ReadWriteOnce"
  PASS=false
fi

# Storage
SIZE=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.spec.resources.requests.storage}' 2>/dev/null)
if [ "$SIZE" = "250Mi" ]; then
  echo "✅ Storage size correct"
else
  echo "❌ Storage request is not 250Mi"
  PASS=false
fi

# PVC Bound
STATUS=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$STATUS" = "Bound" ]; then
  echo "✅ PVC Bound"
else
  echo "❌ PVC not Bound"
  PASS=false
fi

# Deployment exists
if kubectl get deployment mariadb -n mariadb &>/dev/null; then
  echo "✅ Deployment exists"
else
  echo "❌ Deployment not found"
  PASS=false
fi

# Deployment stable
AVAILABLE=$(kubectl get deployment mariadb -n mariadb -o jsonpath='{.status.availableReplicas}' 2>/dev/null)
if [ "$AVAILABLE" = "1" ]; then
  echo "✅ Deployment stable"
else
  echo "❌ Deployment not stable"
  PASS=false
fi

# Deployment uses correct PVC
CLAIM=$(kubectl get deployment mariadb -n mariadb -o jsonpath='{.spec.template.spec.volumes[?(@.persistentVolumeClaim)].persistentVolumeClaim.claimName}' 2>/dev/null)
if [ "$CLAIM" = "mariadb" ]; then
  echo "✅ Deployment using correct PVC"
else
  echo "❌ Deployment not using PVC 'mariadb'"
  PASS=false
fi

echo "--------------------------------------"
if [ "$PASS" = true ]; then
  echo "🎉 CKA TASK SUCCESSFULLY COMPLETED"
else
  echo "🚨 VALIDATION FAILED — REVIEW YOUR FIX"
fi
echo "--------------------------------------"
```
