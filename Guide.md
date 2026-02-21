# CKA Exam Experience: Passed @ 83%

---

## 👨‍💻 Background & Strategy

* **Role:** DevOps Engineer (EKS, AKS, GCP experience).
* **Preparation:** 3 months (~2 hours/day).
* **Goal:** Testing limits on speed and precision under pressure.

---

## 📚 Resources That Worked

| Resource | Why it helps |
| --- | --- |
| **Mumshad’s Udemy Course** | Provides a solid conceptual foundation. |
| **Killercoda.com** | **Critical.** The terminal environment is identical to the actual exam. |
| **Dumbitguy (YouTube)** | Great for supplementary visual learning. |
| **Official K8s Docs** | The only "book" you need. Focus on navigation speed. |

---

## 💡 The "Aha!" Moments

* **Imperative Commands:** Mastery of `kubectl run`, `kubectl expose`, and `kubectl edit` is essential for saving time.
* **Docs Navigation:** Don't memorize YAML. Instead, memorize the path to find snippets.
* *Example:* `Tasks -> Configure Pods and Containers -> Configure a Pod to Use a ConfigMap`.



---

## 🛠 Exam Breakdown & Tough Topics

### 1. Storage

* Managing **PV**, **PVC**, and **StorageClass**.
* Patching an existing StorageClass to set it as `default`.

### 2. Networking

* **NetworkPolicy:** Standard requirement.
* **Gateway API & HTTPRoute:** A modern topic—ensure you study this.

### 3. Operations & Troubleshooting

* **CNI:** Installing a Container Network Interface.
* **Cluster Fixes:** Troubleshooting a broken API server.
* **Helm:** Understanding flags like `--set install.CRDs=true` vs `--skip-crds`.

### 4. Workloads & Configuration

* **Workloads:** Sidecar containers, Resource Quotas, HPA, and patching **PriorityClass**.
* **Configuration:** Custom Resource Definitions (**CRD**), cert-manager, and configuring TLS versions in **ConfigMaps**.

---

## ⏱ Practical Exam Tips

* **Flag Questions:** If a solution isn't verbatim in the docs, understand the *intent* to craft the right command.
* **The Notepad:** Use it to track cluster names and contexts for each question. **Never** work in the wrong context.
* **Pacing:** If a task takes >3 minutes on the first attempt, skip and flag it.

---

## ⌨️ CKA Imperative Commands Cheat Sheet

Mastering these commands allows you to generate YAML templates or modify resources instantly without searching the documentation.

---

### 1. Pod Management

| Task | Command |
| --- | --- |
| **Basic Pod** | `kubectl run nginx --image=nginx` |
| **Dry Run (YAML)** | `kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml` |
| **Pod with Labels** | `kubectl run redis --image=redis --labels="tier=db,app=cache"` |
| **Override Command** | `kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "sleep 3600"` |

---

### 2. Service Exposure

| Task | Command |
| --- | --- |
| **Expose Pod (ClusterIP)** | `kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-service` |
| **Expose Pod (NodePort)** | `kubectl expose pod nginx --port=80 --type=NodePort --name=nginx-np` |
| **Create Service (No Pod)** | `kubectl create service clusterip my-svc --tcp=5678:8080` |

---

### 3. Deployments & Scaling

| Task | Command |
| --- | --- |
| **Create Deployment** | `kubectl create deployment web-app --image=nginx --replicas=3` |
| **Scale Deployment** | `kubectl scale deployment web-app --replicas=5` |
| **Update Image** | `kubectl set image deployment/web-app nginx=nginx:1.21` |
| **Check Rollout** | `kubectl rollout status deployment/web-app` |

---

### 4. Configuration & Security

| Task | Command |
| --- | --- |
| **ConfigMap (Literal)** | `kubectl create configmap my-config --from-literal=key1=val1` |
| **Secret (Literal)** | `kubectl create secret generic my-secret --from-literal=pass=1234` |
| **Resource Quota** | `kubectl create quota my-rq --hard=cpu=1,memory=1G,pods=2` |
| **Service Account** | `kubectl create serviceaccount build-robot` |

---

### 5. Troubleshooting & Maintenance

| Task | Command |
| --- | --- |
| **Temporary Debug Pod** | `kubectl run debug --image=busybox -it --rm -- restart=Never -- sh` |
| **Drain Node** | `kubectl drain node-01 --ignore-daemonsets --force` |
| **Mark Schedulable** | `kubectl uncordon node-01` |
| **View Resource Usage** | `kubectl top pod` or `kubectl top node` |

---

### 💡 Speed Pro-Tip: The Alias

Add this to your shell at the start of the exam to save hundreds of keystrokes:

```bash
alias k=kubectl
complete -F __start_kubectl k # Enables autocomplete for the alias
export do="--dry-run=client -o yaml"

```

*Now you can simply run: `k run nginx --image=nginx $do > pod.yaml*`

## 🛡️ NetworkPolicy & RBAC Cheat Sheet

For the CKA, **NetworkPolicies** and **RBAC** are high-value topics. These commands and snippets help you build security rules quickly without getting lost in the documentation.

---

### 1. Network Policies (Traffic Control)

Network policies are almost always easier to copy from the [Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/network-policies/) and edit. However, you must understand the **selectors**.

**Common Scenarios:**

* **Isolate a Pod:** Deny all ingress/egress by applying a policy with an empty `podSelector`.
* **Allow from Specific Namespace:**
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        project: myproject

```


* **Allow on Specific Port:**
```yaml
ports:
- protocol: TCP
  port: 6379

```



---

### 2. RBAC (Role-Based Access Control)

RBAC is highly imperative-friendly. You can create almost the entire setup using `kubectl create`.

#### A. Creating Roles & ClusterRoles

| Task | Command |
| --- | --- |
| **Role (Namespace)** | `k create role pod-reader --verb=get,list,watch --resource=pods` |
| **ClusterRole** | `k create clusterrole node-reader --verb=get --resource=nodes` |
| **Multiple Resources** | `k create role dev-flat --verb=get --resource=pods,svc,configmaps` |

#### B. Binding Roles to Users/ServiceAccounts

| Task | Command |
| --- | --- |
| **RoleBinding** | `k create rolebinding bob-admin --role=admin --user=bob` |
| **To ServiceAccount** | `k create rolebinding sa-view --clusterrole=view --serviceaccount=default:my-sa` |
| **ClusterRoleBinding** | `k create clusterrolebinding root-node --clusterrole=node-reader --user=root` |

---

### 3. Verification (The "Can-I" Command)

This is the most important troubleshooting tool during the exam to verify your RBAC settings.

| Task | Command |
| --- | --- |
| **Check Current User** | `kubectl auth can-i create deployments` |
| **Check Specific User** | `kubectl auth can-i list pods --as jane` |
| **Check ServiceAccount** | `kubectl auth can-i get secrets --as system:serviceaccount:default:my-sa` |
| **In a Namespace** | `kubectl auth can-i list services -n test-ns --as dev-user` |

---

### 💡 CKA Exam Tip: Labels are Key

Both NetworkPolicies and RBAC (when using `resourceNames`) rely on labels or specific names. Always run `k get pods --show-labels` before writing your YAML to ensure your `podSelector` or `namespaceSelector` matches exactly what is in the cluster.

## 🏁 Final Reflection

> "The time pressure is real. I probably over-engineered a solution or two early on. The key is steady, systematic clicks, not perfection. It feels damn good to finally achieve this feat."
