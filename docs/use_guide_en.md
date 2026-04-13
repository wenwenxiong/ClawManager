[<- Back to README](../README.md)

# ClawManager Deployment and Quick Start Guide

## Table of Contents
- [I. Environment and Goals](#sec-01)
- [II. Deployment Options Overview](#sec-02)
- [III. Option A: Deploy with k3s](#sec-03)
- [IV. Option B: Deploy with Standard Kubernetes](#sec-04)
- [V. Recommendations for Image Pulling on Mainland China Networks (Optional)](#sec-05)
- [VI. Deploy ClawManager](#sec-06)
- [VII. Launch the Web Page](#sec-08)
- [VIII. Quick Start Guide (Initialize and Create an OpenClaw Instance After Login)](#sec-09)
- [IX. Console and Other AI Gateway Features](#sec-12)
- [X. Workspace Module Guide](#sec-13)
- [XI. Quick Troubleshooting Reference](#sec-14)
- [XII. Recommended Final Check Sequence (Use as a Self-Check)](#sec-15)

<a id="sec-01"></a>
## I. Environment and Goals
- **System assumption**: `x86_64` Linux server.
- **Deployment goal**: Deploy **ClawManager**, complete secure model configuration in the Web UI, and then create and start an **OpenClaw Desktop** instance.
- **Applicable scenarios**:
  - **Option A: k3s single-node/lightweight cluster deployment**
  - **Option B: standard Kubernetes cluster deployment** (such as kubeadm clusters, enterprise Kubernetes clusters, and cloud-hosted Kubernetes clusters)


---

<a id="sec-02"></a>
## II. Deployment Options Overview
You can deploy using either of the following methods:

### Option A: k3s deployment
Suitable for single-node, test, or lightweight production environments.

### Option B: standard Kubernetes deployment
Suitable for server environments that already have a standard Kubernetes cluster.

No matter which method you use, you will ultimately apply the same ClawManager manifest:

```bash
kubectl apply -f deployments/k8s/clawmanager.yaml
```

---

<a id="sec-03"></a>
## III. Option A: Deploy with k3s

### 3.1 Install k3s
```bash
curl -sfL https://get.k3s.io | sh -
```

For mainland China networks, you can install using a mirror source:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh |   INSTALL_K3S_MIRROR=cn sh -
```

### 3.2 Check service status
```bash
sudo systemctl status k3s --no-pager
sudo systemctl enable k3s
```

### 3.3 Configure kubectl
If the current user cannot use `kubectl` directly, run:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
```

Or set it temporarily:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 3.4 Verify the cluster
```bash
kubectl get nodes
```

Normally, you should see the node in the `Ready` state.

---

<a id="sec-04"></a>
## IV. Option B: Deploy with Standard Kubernetes

> Applies to x86 server environments that already have an available Kubernetes cluster.

### 4.1 Prerequisite checks
Confirm that the current `kubectl` is connected to the target cluster:

```bash
kubectl get nodes
kubectl get ns
```

Normally, you should see at least one `Ready` node.

### 4.2 Check the default StorageClass
MySQL and MinIO in ClawManager require persistent storage. It is recommended to first check whether the cluster has a default `StorageClass`:

```bash
kubectl get storageclass
```

If the cluster already has a default storage class, you can continue with deployment directly.

If there is **no default StorageClass**, it is recommended to prepare available PV / PVC resources or use a local path storage solution in advance; otherwise, you may later encounter:

```text
pod has unbound immediate PersistentVolumeClaims
```

---

<a id="sec-05"></a>
## V. Recommendations for Image Pulling on Mainland China Networks (Optional)
If the server accesses Docker Hub or other public registries slowly, you can configure image acceleration.

### 5.1 k3s scenario: configure `/etc/rancher/k3s/registries.yaml`
```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
      - "https://docker.nju.edu.cn"
      - "https://docker.1ms.run"
  quay.io:
    endpoint:
      - "https://quay.mirrors.ustc.edu.cn"
  gcr.io:
    endpoint:
      - "https://gcr.mirrors.ustc.edu.cn"
  k8s.gcr.io:
    endpoint:
      - "https://registry.aliyuncs.com/google_containers"
```

After modifying it, run:

```bash
sudo systemctl restart k3s
```

### 5.2 Verify image pulling
```bash
sudo k3s crictl pull docker.io/rancher/mirrored-pause:3.6
```

---

<a id="sec-06"></a>
## VI. Deploy ClawManager

### 6.1 Pull the project code
```bash
git clone https://github.com/Yuan-lab-LLM/ClawManager.git
cd ClawManager
```

### 6.2 Apply the deployment manifest
Run in the repository root directory:

```bash
kubectl apply -f deployments/k8s/clawmanager.yaml
```

### 6.3 Check base resources
```bash
kubectl get ns
kubectl get pods -n clawmanager-system
kubectl get svc -n clawmanager-system
```

Under normal circumstances, you will see the following components:
- `clawmanager-app`
- `mysql`
- `minio`
- `skill-scanner`

If you see the following error:

```text
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

it means MySQL / MinIO in cluster storage cannot start because the PVC is not bound. Please jump directly to the end of this document:

- [XI.1 Dedicated Handling for Storage Issues (PV/PVC)](#sec-14-storage)

---

<a id="sec-08"></a>
## VII. Launch the Web Page

### 7.1 Access via NodePort
By default, the ClawManager frontend Service uses an HTTPS NodePort. You can check it first:

```bash
kubectl get svc -n clawmanager-system
```

If the frontend port is:

```text
443:30443/TCP
```

you can access it directly in the browser:

```text
https://<serverIP>:30443
```


### 7.2 First HTTPS access note
Since it usually uses a self-signed certificate, the browser may show an “unsafe” or certificate warning. Click:

```text
Advanced → Continue to visit
```

to enter the page.

---

<a id="sec-09"></a>
## VIII. Quick Start Guide (Initialize and Create an OpenClaw Instance After Login)

After completing the deployment above and successfully opening the management page, you still need to finish the following initialization steps before you can actually create and start an **OpenClaw** instance.

### 8.1 Log in to the system
1. Open the deployed page, for example: `https://<nodeIP>:30443`.
2. Log in with the default administrator account:
   - **Username**: `admin`
   - **Password**: `admin123`
3. After first login, it is recommended to change the default password as needed.


### 8.2 Configure the secure model (AI Gateway)

![Figure 1: AI Gateway configuration](./main/1.png)
After logging in, you need to configure an available **secure model** first so that it can be used uniformly by the platform and subsequent instances.

1. Click the left-side menu: **AI Gateway** → **Models**.
2. Add a new model or edit an existing one, and fill in the following information according to the actual model service you connect:

   * **Display Name**: Enter a name that is easy to identify.
   * **Vendor Template**: Choose the corresponding template based on your model service type; if you use a custom or compatible interface, you can select **Local / Internal**.
   * **Protocol**: Select the protocol according to the interface, such as **OpenAI Compatible** or another actual protocol.
   * **Base URL**: Enter the endpoint address provided by the model service.
   * **API Key**: Enter the valid key for the corresponding model service.
   * **Provider Model**: Enter the actual model name to call.
   * **Currency**: Fill it in according to your situation; if no billing display is needed, you can keep the default.
   * **Input Price / Output Price**: If billing statistics are not needed, you can first fill in `0`.
3. Be sure to check the following before submission:

   * **Secure Model**
   * **Enabled**
4. Click **Save**.

> Note: The images on the page are only used to show the field positions and example format. The actual content should be based on the model service configuration you use.


### 8.3 Create an OpenClaw instance
After the model configuration is completed, create an **OpenClaw Desktop** instance.

1. Click **ADMIN** in the lower-left corner and switch to the **Workspace**.
2. Click **Create Instance**.

![](./main/2.png)
#### Step 1: Basic Information
- Fill in the **Instance Name** (at least 3 characters).
- The description is optional and may be left blank.
- Click **Next**.

![](./main/3.png)
#### Step 2: Select Type
- Select **OpenClaw Desktop**.
- Click **Next**.


![](./main/4.png)
#### Step 3: Configuration
- You can directly choose the **Small** specification:
  - `2 CPU`
  - `4 GB RAM`
  - `20 GB Disk`
- You can also modify the settings as needed in the custom configuration section below.
- For the OpenClaw resource injection section, you can choose as needed:
  - **Manual Resources**
  - **Resource Bundle**
  - **Archive Import**
- For first-time use, you can keep the default or select **Manual Resources**.
- Finally, click **Create**.

### 8.4 First creation note
- When creating an **OpenClaw** instance for the first time, the required images must be downloaded and the environment must be initialized, so it will take noticeably longer.
- On slow networks or during the first image pull, the instance status may remain at **Creating** for a long time. Please wait patiently.
- If it still does not start successfully after a long time, go back to the Kubernetes / Docker logs to troubleshoot image, PVC, gateway model, and other issues.

---

<a id="sec-12"></a>
## IX. Console and Other AI Gateway Features

In addition to model configuration, the platform homepage console and the AI Gateway also provide auditing, cost, and rule governance features, making it easier for administrators to centrally view cluster status, model call records, and security policy execution status.

### 9.1 Console Overview

![](./main/5.png)

The console homepage is used to display the overall running status of the current cluster and platform, allowing administrators to quickly understand resource usage and system health.

It mainly includes the following information:

- **Cluster Basic Information Overview**: Displays the total number of users, total number of instances, number of running instances, and total storage usage of the current platform.
- **Node Overview**: Displays the current number of available nodes, as well as the main scheduling node information in the current cluster.
- **Resource Request Status**: Displays the total amount of CPU, memory, and disk resources that have been requested by the current platform.
- **Capacity Dashboard**: Shows overall resource capacity and current usage rates by node, CPU, memory, disk, and other dimensions, making it easier to determine whether the cluster still has available capacity.
- **Infrastructure Table**: Used to view the status information of current nodes, resources, and the basic runtime environment.

> Note: The console is mainly used to view the overall platform resources, nodes, and instance operation summary, and is not used directly for specific OpenClaw operations inside an instance.

### 9.2 Overview of AI Gateway Features

In addition to **Models**, the AI Gateway also includes the following modules:

- **AI Audit**: View model call traces, request and response payloads, risk hits, routing decisions, and call details.
- **Cost**: View token usage, estimated costs, internal costs, and trend statistics.
- **Risk Control Rules**: Configure sensitive content detection rules and control whether matched content is allowed through or routed to the secure model.

### 9.3 Cost Module

The Cost page is used to count the costs and token usage of platform model calls, helping administrators understand the overall consumption.

![](./main/6.png)

The page mainly includes the following contents:

- **Input Tokens**: Counts the total amount of input prompts.
- **Output Tokens**: Counts the total amount of model-generated content.
- **Estimated Cost**: Cost estimated according to Provider pricing.
- **Internal Cost**: Internal accounting cost related to the secure model.
- **Daily Cost Trend**: View estimated cost and token changes in the current window over the last 7 days.
- **User Summary**: Aggregate usage and cost by user.
- **Instance Summary**: Aggregate usage and cost by instance.
- **Recent Cost Records**: Supports searching and paging through cost records by Trace, user, model, and other conditions, and supports jumping to audit details.

> Note: If no model call records have been generated yet, input tokens, output tokens, costs, and trend charts may all be 0, which is normal.

### 9.4 AI Audit Module

The AI Audit page is used to view recent managed model call records and helps administrators troubleshoot model calls, token usage, and routing results.

![](./main/7.png)

Main features include:

- **Recent AI Traces**: View recent model call paths.
- **Trace List**: View recent managed traces in a unified table.
- **Search and Filter**: Supports searching by trace, request content, user, model, and other conditions.
- **Status Filter**: Supports viewing different call results by status.
- **Model Filter**: Supports filtering corresponding call records by model.
- **Paged Refresh**: Supports paginated viewing and manually refreshing the latest audit results.

> Note: If the page shows “No AI audit records”, it means that no actual model invocation request has been generated yet.

### 9.5 Risk Control Rules Module

The Risk Control Rules page is used to configure sensitive content detection rules and decide the handling action after a rule match.

![](./main/8.png)

This module mainly supports:

- **Rule List Management**: View all rules and their enabled status.
- **Rule Category Viewing**: Supports viewing rules by categories such as personal information, company information, customer business, security credentials, finance and legal, politically sensitive, custom, and others.
- **Rule Field Configuration**: You can set the rule ID, display name, severity level, action, sort order, regex pattern, and description.
- **Rule Action Control**: After a rule is matched, you can choose to allow it through or route it to the secure model.
- **Bulk Enable / Disable**: Supports bulk adjustment of rule status.
- **Rule Testing Console**: You can paste sample text to test which enabled rules or draft rules will be matched.

Built-in rule examples currently include but are not limited to:

- Personal information: email address, mobile number, ID card number, passport number, bank card context, address, resume content, etc.
- Company information: private network IP, internal domain name, host naming, Kubernetes Service DNS, project codename, organizational structure, salary / HR information, etc.
- Customer business: customer lists, contracts / quotations, invoice tax numbers, CRM / ticket data, etc.
- Security credentials: private keys, API keys, tokens, JWTs, Cookie / Session, database connection strings, kubeconfig, environment variable secrets, etc.
- Finance and legal: budgets, profits, revenue, legal opinions, litigation, NDAs, etc.
- Politically sensitive: political institutions, military/national security, extremist violence-related expressions, etc.

> Note: The default rules already cover many common sensitive information detection scenarios. In actual use, you can continue to add, adjust, or disable some rules according to business needs.
---

<a id="sec-13"></a>
## X. Workspace Module Guide

The Workspace is the main operating area after a regular user enters the platform. It is used to view personal resource quotas, create instances, manage instances, and maintain OpenClaw-related resources. This module is more oriented toward daily use and operations than the administrator-side “Console Overview”.

### 10.1 Workspace Home
![](./main/9.png)
The Workspace home page is used to display the instance and resource usage summary of the current account, and mainly includes the following contents:

- **My Instances**: Displays the number of instances created under the current account.
- **Running**: Displays the number of instances currently running.
- **Used Storage**: Displays the amount of storage space currently occupied by the account.
- **My Resource Quotas**: Shows the available quota information of the current account, including the number of instances, maximum CPU cores, maximum memory, maximum storage, and maximum GPU count.
- **Quick Actions**: Provides two entry points: **Create New Instance** and **View All Instances**, so you can get started quickly with the platform.

> Note: When the page shows “No instances yet”, you can directly click **Create New Instance** to start creating the first OpenClaw Desktop instance.

### 10.2 My Instances

The **My Instances** page is used to centrally view and manage all instances created under the current account. This page mainly carries the instance management functions.
![](./main/10.png)
Common supported operations include:

- **View instance status**: Check whether the instance is being created, running, stopped, or in an abnormal state.
- **Open instance details**: View basic instance information, resource configuration, and runtime status.
- **Stop instance**: When the instance is abnormal or the environment needs to be reloaded, you can perform a stop operation.
- **Delete instance**: When the instance is no longer needed, you can delete it directly to release the corresponding CPU, memory, and storage resources.

> Note: After deleting an instance, the related resources of the instance will be cleaned up together. Before executing, make sure that the data and configuration inside it have been backed up.

### 10.3 Resource Management

The **Resource Management** page is used to maintain the OpenClaw resource content available for use, making it easy to inject and use after an instance starts.
![](./main/11.png)
The page mainly includes the following parts:

- **Resources**: View and maintain available resource entries.
- **Resource Bundles**: Combine multiple resources into reusable bundles to facilitate batch injection.
- **Injection Records**: View resource injection history and execution status.

On the left side of the Resource Management page, you can also manage resources by type. The currently visible types on the page include:

- **Channels**
- **Skills**
- **Agents (coming soon)**
- **Scheduled Tasks (coming soon)**

The upper-right corner of the page supports:

- **Refresh**: Reload the current resource list.
- **New**: Create a new resource item.

> Note: Resource Management is mainly used to prepare OpenClaw resource content that can be used after the instance starts, and does not directly replace the instance creation process. When creating an instance, resources can be injected through methods such as **Manual Resources**, **Resource Bundles**, and **Archive Import**.

---

<a id="sec-14"></a>
## XI. Quick Troubleshooting Reference

<a id="sec-14-storage"></a>
### 11.1 Dedicated Handling for Storage Issues (PV/PVC)

If you see the following error:

```text
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

it means the cluster storage was not bound automatically. In this case, you can manually create local `hostPath` PV/PVC in the x86 single-node server style.

> This solution is suitable for single-node server testing or lightweight environments. For production environments, it is recommended to use formal storage such as NFS, Ceph, or cloud disks instead.

#### 11.1.1 Create PV
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-local
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /tmp/mysql-data
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minio-pv-local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /tmp/minio-data
EOF
```

#### 11.1.2 Create PVC
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: clawmanager-system
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: mysql-pv-local
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data
  namespace: clawmanager-system
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: minio-pv-local
EOF
```

#### 11.1.3 Recreate Pod
```bash
kubectl delete pod --all -n clawmanager-system
```

#### 11.1.4 Observe status again
```bash
kubectl get pvc -n clawmanager-system
kubectl get pods -n clawmanager-system -w
```

Expected results:
- `mysql-data` / `minio-data` are `Bound`
- `mysql` / `minio` / `skill-scanner` / `clawmanager-app` are finally `Running`

---

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| `kubectl` connection to `localhost:8080` is refused | kubeconfig is not configured | Set `KUBECONFIG` or copy it to `~/.kube/config` |
| Pod image pull timeout | Network to Docker Hub / GHCR is unstable | Configure image acceleration or a proxy |
| MySQL / MinIO remain `Pending` | PVC not bound | Check the `StorageClass` or manually create PV/PVC |
| The browser cannot open the page | NodePort is not open / the `port-forward` process was not kept running | Open the port or keep the forwarding terminal running |
| The page opens but an OpenClaw instance cannot be created | Secure model is not configured | First configure and enable the secure model under **AI Gateway → Models** |
| The instance remains “Creating” for a long time | The first image pull takes a long time / storage or network issues | Wait patiently, and if necessary check Pods and events |

---

<a id="sec-15"></a>
## XII. Recommended Final Check Sequence (Use as a Self-Check)
1. `kubectl get nodes`
2. `kubectl get storageclass`
3. `kubectl get pods -n clawmanager-system`
4. `kubectl get pvc -n clawmanager-system`
5. `kubectl get svc -n clawmanager-system`
6. Open `https://<IP>:30443` in a browser
7. Log in to the backend and complete **secure model configuration**
8. Create an **OpenClaw Desktop** instance in the Workspace
