# OpenShift Bare-Metal Virtual Loop Disk & TopoLVM Setup

A complete guide and reference for simulating block storage on bare-metal OpenShift cluster nodes using file-backed virtual loop devices and configuring the **OpenShift LVM Storage Operator (TopoLVM)**.

---

## 📋 Table of Contents

1. [Why Use a Virtual Loop Disk?](#-why-use-a-virtual-loop-disk)
2. [Where Can It Be Used?](#-where-can-it-be-used)
3. [Required Operator](#-required-operator)
4. [Step-by-Step Procedure](#-step-by-step-procedure)

---

## 💡 Why Use a Virtual Loop Disk?

In bare-metal infrastructure, dedicated physical disks or secondary SSDs are not always available. Virtual loop disks provide:

* **Hardware Independence:** Simulate a raw block device using a sparse file on the host's existing filesystem.
* **Strict Prerequisite Compliance:** Satisfy operators and services (like local container registries or databases) that explicitly demand dedicated block storage or Volume Groups.
* **Rapid Provisioning:** Easily scale, resize, or clean up storage volumes without physical hardware modifications.

---

## 🌐 Where Can It Be Used?

* **Bare-Metal Clusters:** Nodes equipped with only a single primary OS drive.
* **Edge & Single-Node OpenShift (SNO):** Compact environments with constrained physical expansion slots.
* **Dev/Test & CI/CD Labs:** Virtualized or minimal hardware footprints mimicking enterprise environments.
* **Air-Gapped Registries:** Supplying heavy persistent capacity for local mirror registries (e.g., IBM Maximo Application Suite air-gap setups).

---

## ⚙️ Required Operator

This setup relies on the **Red Hat OpenShift LVM Storage Operator (TopoLVM)**.

* **Namespace:** `openshift-storage`
* **Custom Resource (CRD):** `LvmCluster` (`lvmcluster.lvm.topolvm.io`)
* **Function:** It detects the loop device attached to the node, creates a Volume Group (`vg1`), and exposes it as a local storage backend for dynamic persistent volume provisioning.

---

## 🚀 Step-by-Step Procedure

### Step 1: Debug the Target Node

Access the specific bare-metal node using OpenShift's debug utility. Replace `<node-name>` with your actual node identifier (e.g., `00-be-43-80-03-43`).

```bash
oc debug node/<node-name>
```

Once inside the debug pod, switch your root directory context to the underlying host filesystem:

```bash
chroot /host
```

---

### Step 2: Create the Loop Image File

1. Create the target directory structure for your local storage backend:
	```bash
	mkdir -p /var/local/lvms
	```

2. Allocate a sparse file to act as the virtual disk image (e.g., `200GB` capacity):
	```bash
	truncate -s 200G /var/local/lvms/registry-disk.img
	```
---

### Step 3: Configure and Attach the Loop Device

1. Find the next available loop device:
	```bash
	losetup -f
	```

	Example output: `/dev/loop1`

2. Associate the loop device with the sparse disk image file you created:
	```bash
	losetup /dev/loop1 /var/local/lvms/registry-disk.img
	```

3. Verify that the loop device is successfully created and correctly sized:
	```bash
	lsblk /dev/loop1
	```

	Expected output:

	```text
	NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
	loop1    7:1    0  200G  0 loop
	```

4. Exit out of the `chroot` environment and terminate the debug session:
	```bash
	exit
	exit
	```
---

### Step 4: Apply the TopoLVM Cluster Configuration

1. Create or edit your LVM cluster configuration manifest (e.g., `lvm-loop-cluster.yaml`) referencing the newly created loop device on your node.
2. Apply the manifest to provision the volume group (`vg1`):
	```bash
	oc apply -f lvm-loop-cluster.yaml
	```
---

### Step 5: Validate the Volume Group and Node Status

Check that the volume group and node statuses have initialized successfully within the `openshift-storage` namespace:

```bash
oc get lvmvolumegroups -n openshift-storage
oc get lvmvolumegroupnodestatuses -n openshift-storage
```

---
