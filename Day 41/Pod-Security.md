# Pod-Security Scenarios

Assume:

```bash
kubectl run test --image=ubuntu -- sleep 3600
kubectl exec -it test -- bash
```

Now let’s see what breaks depending on security settings.

---

# 🔐 1️⃣ runAsNonRoot: true

| Setting              | Blocks                     | Command That Fails           | Why It Fails                  | Recommended        |
| -------------------- | -------------------------- | ---------------------------- | ----------------------------- | ------------------ |
| `runAsNonRoot: true` | Running container as UID 0 | Pod fails to start           | If image default user is root | Always set to true |
| `runAsUser: 1000`    | Root execution             | `whoami` → root not possible | Forces non-root user          | Use 1000+          |

### Example Failure

If image default is root:

```yaml
runAsNonRoot: true
```

Pod error:

```
container has runAsNonRoot and image will run as root
```

Container never starts.

---

# 🔐 2️⃣ allowPrivilegeEscalation: false

Blocks gaining more privileges via setuid binaries.

| Setting                           | Blocks                      | Command That Fails | Why                |
| --------------------------------- | --------------------------- | ------------------ | ------------------ |
| `allowPrivilegeEscalation: false` | setuid privilege escalation | `sudo su`          | Cannot elevate     |
|                                   |                             | `chmod +s file`    | setuid ineffective |

### Inside Ubuntu

Try:

```bash
sudo su
```

Result:

```
permission denied
```

Or privilege won't increase.

---

# 🔐 3️⃣ privileged: false

If privileged = true → full host access
If false → limited container namespace

| Setting             | Blocks              | Command That Fails    | Why                    |
| ------------------- | ------------------- | --------------------- | ---------------------- |
| `privileged: false` | Access host devices | `mount /dev/sda /mnt` | No device access       |
|                     | Kernel module load  | `modprobe dummy`      | No kernel access       |
|                     | Raw disk access     | `fdisk -l`            | No block device access |

---

# 🔐 4️⃣ capabilities.drop: ALL

Removes Linux capabilities.

| Capability Dropped | Command That Fails              | Why                       |
| ------------------ | ------------------------------- | ------------------------- |
| `NET_ADMIN`        | `ip link add dummy0 type dummy` | Cannot modify network     |
| `SYS_ADMIN`        | `mount -t tmpfs tmpfs /mnt`     | Cannot mount              |
| `SYS_PTRACE`       | `strace -p 1`                   | Cannot trace process      |
| `DAC_OVERRIDE`     | `cat /etc/shadow`               | Cannot bypass permissions |
| `CHOWN`            | `chown root file`               | Cannot change ownership   |

---

### Example

Inside Ubuntu:

```bash
mount -t tmpfs tmpfs /mnt
```

Error:

```
mount: permission denied
```

Because `SYS_ADMIN` dropped.

---

# 🔐 5️⃣ readOnlyRootFilesystem: true

| Setting                        | Blocks              | Command That Fails      | Why                |
| ------------------------------ | ------------------- | ----------------------- | ------------------ |
| `readOnlyRootFilesystem: true` | Writing to root FS  | `touch /file.txt`       | FS is read-only    |
|                                | Installing packages | `apt install vim`       | Needs write access |
|                                | Modifying /etc      | `echo test > /etc/test` | Cannot write       |

### Example

```bash
touch /newfile
```

Error:

```
Read-only file system
```

---

# 🔐 6️⃣ seccompProfile: RuntimeDefault

Blocks dangerous syscalls.

| Setting          | Blocks               | Command That Fails         | Why              |
| ---------------- | -------------------- | -------------------------- | ---------------- |
| `RuntimeDefault` | Uncommon syscalls    | Custom exploit tools       | Syscall filtered |
|                  | ptrace-like behavior | `strace` sometimes blocked | Security filter  |

Advanced exploitation tools fail silently.

---

# 🔐 7️⃣ fsGroup

Controls shared volume permissions.

If not set properly:

```bash
touch /mnt/volume/file
```

Error:

```
Permission denied
```

Because user not in correct group.

---

# 🔐 8️⃣ hostNetwork: false

| Setting              | Blocks                       | Command That Fails                 | Why                 |
| -------------------- | ---------------------------- | ---------------------------------- | ------------------- |
| `hostNetwork: false` | Access to host network stack | `netstat -tulnp` (won’t show host) | Isolated namespace  |
|                      | Binding host ports           | `nc -l 80` may fail                | No host port access |

---

# 🔥 FULL SUMMARY TABLE (CKS-Level View)

| Security Setting                | What It Protects    | Ubuntu Command That Fails | Attack Prevented     |
| ------------------------------- | ------------------- | ------------------------- | -------------------- |
| runAsNonRoot                    | Root container      | Pod won’t start           | Root breakout        |
| allowPrivilegeEscalation: false | sudo/setuid         | `sudo su`                 | Privilege escalation |
| privileged: false               | Host access         | `mount /dev/sda`          | Node takeover        |
| drop: ALL                       | Kernel powers       | `mount`, `ip link add`    | Kernel tampering     |
| readOnlyRootFilesystem          | Malware persistence | `touch /file`             | File tampering       |
| seccomp RuntimeDefault          | Syscall abuse       | Exploit binaries          | Kernel exploit       |
| fsGroup misconfig               | Volume misuse       | `touch /mnt/file`         | Shared volume abuse  |
| hostNetwork: false              | Host network access | Can't see host ports      | Network snooping     |

---

# 🧠 Real Attack Scenario

If you allow:

```yaml
privileged: true
capabilities:
  add:
    - SYS_ADMIN
```

Attacker can run:

```bash
mount /dev/sda1 /mnt
chroot /mnt
```

Now they are inside the node.

Cluster compromised.

---

# 🚀 Production Secure Baseline (Ubuntu container safe)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  seccompProfile:
    type: RuntimeDefault

containers:
- securityContext:
    allowPrivilegeEscalation: false
    privileged: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
        - ALL
```

---

# 🎯 Final Understanding

SecurityContext is NOT theoretical.

It directly blocks:

* mount
* sudo
* kernel operations
* network tampering
* writing to filesystem
* privilege escalation

It reduces container breakout risk drastically.

---
