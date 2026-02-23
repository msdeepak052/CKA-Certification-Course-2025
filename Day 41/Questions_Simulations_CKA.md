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
echo "🔍 CKA VALIDATION STARTED"
echo "--------------------------------------"

PASS=true

echo "1️⃣ Checking namespace..."
kubectl get ns mariadb &>/dev/null || { echo "❌ Namespace mariadb missing"; PASS=false; }

echo "2️⃣ Checking PV..."
PV_COUNT=$(kubectl get pv --no-headers | wc -l)
if [ "$PV_COUNT" -ne 1 ]; then
  echo "❌ There should be exactly ONE PV"
  PASS=false
else
  echo "✅ One PV exists"
fi

echo "3️⃣ Checking PVC existence..."
kubectl get pvc mariadb -n mariadb &>/dev/null || { echo "❌ PVC 'mariadb' not found"; PASS=false; }

echo "4️⃣ Checking PVC AccessMode..."
MODE=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.spec.accessModes[0]}' 2>/dev/null)
if [ "$MODE" != "ReadWriteOnce" ]; then
  echo "❌ PVC AccessMode is not ReadWriteOnce"
  PASS=false
else
  echo "✅ AccessMode correct"
fi

echo "5️⃣ Checking PVC Storage..."
SIZE=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.spec.resources.requests.storage}' 2>/dev/null)
if [ "$SIZE" != "250Mi" ]; then
  echo "❌ PVC storage is not 250Mi"
  PASS=false
else
  echo "✅ Storage size correct"
fi

echo "6️⃣ Checking PVC Bound status..."
STATUS=$(kubectl get pvc mariadb -n mariadb -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$STATUS" != "Bound" ]; then
  echo "❌ PVC is not Bound"
  PASS=false
else
  echo "✅ PVC Bound"
fi

echo "7️⃣ Checking Deployment..."
kubectl get deployment mariadb -n mariadb &>/dev/null || { echo "❌ Deployment not found"; PASS=false; }

echo "8️⃣ Checking Deployment availability..."
AVAILABLE=$(kubectl get deployment mariadb -n mariadb -o jsonpath='{.status.availableReplicas}' 2>/dev/null)
if [ "$AVAILABLE" != "1" ]; then
  echo "❌ Deployment not available"
  PASS=false
else
  echo "✅ Deployment available"
fi

echo "9️⃣ Checking Deployment uses PVC..."
CLAIM=$(kubectl get deploy mariadb -n mariadb -o jsonpath='{.spec.template.spec.volumes[0].persistentVolumeClaim.claimName}' 2>/dev/null)
if [ "$CLAIM" != "mariadb" ]; then
  echo "❌ Deployment is not using PVC 'mariadb'"
  PASS=false
else
  echo "✅ Deployment using correct PVC"
fi

echo "🔟 Checking Data Preservation..."
POD=$(kubectl get pod -n mariadb -l app=mariadb -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -z "$POD" ]; then
  echo "❌ Pod not running"
  PASS=false
else
  DB_CHECK=$(kubectl exec -n mariadb $POD -- mysql -uroot -prootpass -e "SHOW DATABASES;" 2>/dev/null | grep cka_test)
  if [ -z "$DB_CHECK" ]; then
    echo "❌ Database cka_test not found (Data NOT preserved)"
    PASS=false
  else
    echo "✅ Data preserved (cka_test exists)"
  fi
fi

echo "--------------------------------------"
if [ "$PASS" = true ]; then
  echo "🎉 ALL CHECKS PASSED — CKA TASK SOLVED CORRECTLY"
else
  echo "🚨 SOME CHECKS FAILED — REVIEW YOUR SOLUTION"
fi
echo "--------------------------------------"
```
