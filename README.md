
# Kubernetes Control-Plane Node Configuration for Fedora CoreOS

This document outlines the configuration for provisioning a single-node Kubernetes cluster on a Fedora CoreOS machine using Butane. The resulting node serves as both the control-plane and a worker, making it a complete, self-contained cluster.

## Overview

This Butane configuration automates the entire setup process, including:
*   Setting up user access and networking.
*   Installing the necessary Kubernetes and container runtime packages.
*   Configuring the kernel and system services.
*   Initializing the Kubernetes cluster with `kubeadm`.
*   Automating post-installation steps for immediate usability.

## Usage

This is a Butane source file (`.bu`). It must be compiled into an Ignition file (`.ign`) before being provided to a Fedora CoreOS machine during its first boot.

---

## Component Breakdown

### `passwd`

This section configures user access.

*   **`users`**: Defines a user named `core`.
*   **`ssh_authorized_keys`**: Grants SSH access to the `core` user by providing a public SSH key.
*   **`groups`**: Adds the `core` user to the `wheel` group, which grants `sudo` privileges.

### `storage`

This section creates all the necessary configuration files on the node's filesystem.

*   **/etc/hostname**: Sets the machine's hostname to `k8s-node`.
*   **/etc/yum.repos.d/**: Adds the official package repositories for Kubernetes (`kubernetes.repo`) and the CRI-O container runtime (`cri-o.repo`). This allows the system to find and install these packages.
*   **/etc/modules-load.d/k8s.conf**: Ensures the `overlay` and `br_netfilter` kernel modules are loaded on boot. These are required by the container runtime and for Kubernetes networking.
*   **/etc/sysctl.d/k8s.conf**: Configures necessary kernel parameters for Kubernetes networking, specifically enabling IP forwarding and allowing `iptables` to see bridged traffic.
*   **/etc/kubernetes/kubeadm-config.yaml**: This is the central configuration for the `kubeadm` tool.
    *   **`KubeletConfiguration`**: Configures the `kubelet` to use the `systemd` cgroup driver, which is essential for proper resource management.
    *   **`InitConfiguration`**: Tells `kubeadm` that the container runtime is CRI-O, specified via its socket path.
    *   **`ClusterConfiguration`**: Defines cluster-wide settings, most importantly the `podSubnet` (`10.244.0.0/16`), which is the default network range for the Flannel CNI plugin.
*   **/usr/local/bin/kubeadm-post-init.sh**: A helper script that automates all post-installation tasks. Placing it in `/usr/local/bin` ensures it has the correct SELinux context to be executed by `systemd`. The script performs the following:
    1.  Creates a `.kube` directory for the `core` user.
    2.  Copies the `admin.conf` file so the `core` user can immediately use `kubectl` on the node.
    3.  Removes the `control-plane` taint from the node, allowing it to function as a worker and run application pods.
    4.  Copies `admin.conf` to the `core` user's home directory for easy downloading with `scp`.

### `systemd`

This section defines and enables the `systemd` services that orchestrate the entire setup process.

*   **`install-k8s.service`**:
    *   **Purpose**: Runs **only on the first boot**.
    *   **Action**: Uses `rpm-ostree` to install the `cri-o`, `kubelet`, `kubeadm`, and `kubectl` packages.
    *   **Key Feature**: After the packages are installed, the Fedora CoreOS system **automatically reboots** to activate the new software deployment. A stamp file (`/var/lib/install-k8s.service.stamp`) is created to prevent this service from ever running again.
*   **`crio.service` & `kubelet.service`**:
    *   **Purpose**: Enables the container runtime and the Kubernetes node agent.
    *   **Action**: Ensures these services are started automatically on boot (after the first-boot reboot).
*   **`kubeadm-init.service`**:
    *   **Purpose**: The main orchestrator that runs on the **second boot**.
    *   **Action**:
        1.  It `Requires` `crio.service` to be running and `Wants` `kubelet.service` to have started.
        2.  It runs `kubeadm init` using the `/etc/kubernetes/kubeadm-config.yaml` file to bootstrap the Kubernetes cluster.
        3.  Upon successful completion, it executes the `/usr/local/bin/kubeadm-post-init.sh` script to finalize the setup.

---

## Post-Provisioning Steps

After the machine has been provisioned with this configuration and has finished rebooting, the cluster will be up, but networking will not be active for pods.

1.  **Install a CNI Plugin**: The pods will be stuck in a `Pending` state until a network plugin is installed. To install Flannel (which this configuration is set up for), run:

2.  **Access the Cluster Remotely**: To manage the cluster from your development machine, copy the `admin.conf` file from the node, update the server IP address, and use it with your local `kubectl`.


---

## Convert Butane file to Ignition (.ign)

```shell
docker run --interactive --rm --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release --pretty --strict k8s-control-plane.yml > coreos.ign; $content = Get-Content -Path coreos.ign -Raw; [System.IO.File]::WriteAllText("coreos.ign", $content, (New-Object System.Text.UTF8Encoding($false)))
```

## Install and serve Ignition file

**On your local machine**,
```shell
python3 -m http.server 8000
```

**On your VM machine**
```shell
sudo coreos-installer install /dev/sda --ignition-url=<url_ign_file> --insecure-ignition
```

---

## K8S

### crio.service
[CRI-O](https://cri-o.io/) is the engine that runs your containers, and the CRI is the standard language it uses to take orders from Kubernetes.

**How we use it:** We install it on the node and tell the `kubelet` to talk to it by pointing to its communication file (the "socket" at /var/run/crio/crio.sock)


### Accessing the Cluster Remotely

To manage your new Kubernetes cluster from your local development machine, you need to copy the cluster's configuration file (`admin.conf`), which contains the necessary credentials and server address.

#### 1. Download the Cluster's Kubeconfig

The post-install script on the server conveniently placed a copy of the `admin.conf` file in the `core` user's home directory.

**On your local machine**, use `scp` to download this file and save it as your default kubeconfig:

```shell
scp core@<VM_IP>:/home/core/admin.conf $HOME/.kube/config
```


#### 2. Update the Server IP Address in the Kubeconfig

The downloaded file is configured to be used *on the server itself*, so it points to an internal address. You must update it to point to your VM's public IP address.

1.  Open the `$HOME/.kube/config` file on your local machine in a text editor.
2.  Find the `server:` line. It will look something like this:
3.
```yaml
YAMLclusters:
- cluster:
  certificate-authority-data: <REDACTED_DATA>
  server: https://10.0.2.15:6443  # <-- CHANGE TO YOU YOUR VM'S IP ADDRESS
  name: kubernetes3.Change the IP address in the server: field to your VM's actual, reachable IP address.YAMLclusters:
- cluster:
```

#### 3. Verify the Connection

You should now be able to connect to your cluster from your local machine.

```shell
kubectl get nodes
```