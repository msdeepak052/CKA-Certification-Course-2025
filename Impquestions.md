
## Task

This task must be completed on **node01**. Use the following credentials to access the node via SSH:

* **Username:** `bob`
* **Password:** `caleston123`

### Objective

Prepare **node01** for a Kubernetes installation by setting up a container runtime.

1. Install the `cri-docker_0.3.16.3-0.debian.deb` package located in `/root`.
2. Ensure the `cri-docker` service is **running** and **enabled** to start on boot.

---

## Solution

### 1. Access the Node

SSH into **node01** using the provided password:

```bash
ssh bob@node01

```

### 2. Elevate Privileges

Switch to the root user to execute administrative commands:

```bash
sudo -i

```

### 3. Install the Package

Use `dpkg` to install the `.deb` package located in the `/root` directory:

```bash
dpkg -i /root/cri-docker_0.3.16.3-0.debian.deb

```

### 4. Manage the Service

Start the `cri-docker` service and configure it to launch automatically at boot:

```bash
systemctl start cri-docker
systemctl enable cri-docker

```

### 5. Verification

Confirm the service is active and properly enabled:

| Command | Expected Output |
| --- | --- |
| `systemctl is-active cri-docker` | `active` |
| `systemctl is-enabled cri-docker` | `enabled` |

---
