# Kubernetes Control-Plane Node Configuration for Fedora CoreOS

This document outlines the configuration for provisioning a single-node Kubernetes cluster on a Fedora CoreOS machine using Butane. The resulting node serves as both the control-plane and a worker, making it a complete, self-contained cluster.

## Overview

This Butane configuration automates the entire setup process, including:
*   Setting up user access and networking.
*   Installing the necessary Kubernetes and container runtime packages.
*   Configuring the kernel and system services.
*   Initializing the Kubernetes cluster with `kubeadm`.
*   Automating post-installation steps for immediate usability.

---

## Post-Provisioning Steps

After the machine has been provisioned and has finished its setup, the cluster will be up and running, but it requires two final steps to become fully operational.

### 1. Install the CNI Network Plugin (Flannel)

After `kubeadm` finishes, you will notice that some pods (like CoreDNS) are stuck in the `Pending` state. This is expected. These pods require a network to communicate, which is provided by a CNI (Container Network Interface) plugin.

This configuration is pre-set to use the default network range for **Flannel**, a simple and popular CNI plugin. For more information on cluster addons like networking and DNS, see the [official Kubernetes documentation](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

**On your local machine**, run the following command to install Flannel:

```shell
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

After a minute, the `kube-flannel` and `coredns` pods will become `Running`, and the cluster network will be fully operational.

### 2. Access the Cluster Remotely

To manage your new Kubernetes cluster from your local development machine, you need to copy the cluster's configuration file (`admin.conf`), which contains the necessary credentials and server address.

#### Download the Cluster's Kubeconfig

The post-install script on the server conveniently placed a copy of the `admin.conf` file in the `core` user's home directory.

**On your local machine**, use `scp` to download this file and save it as your default kubeconfig:

```shell
# Replace <VM_IP> with the actual IP of your Kubernetes node
scp core@<VM_IP>:/home/core/admin.conf $HOME/.kube/config
```

#### Update the Server IP Address in the Kubeconfig

The downloaded file is configured to be used *on the server itself*, so it points to an internal address. You must update it to point to your VM's public IP address.

1.  Open the `$HOME/.kube/config` file on your local machine in a text editor.
2.  Find the `server:` line and change the hostname to your VM's actual, reachable IP address.

```yaml
- cluster:
  server: https://10.0.2.15:6443  # <-- CHANGE TO YOU YOUR VM'S IP ADDRESS
```

#### Verify the Connection

You should now be able to connect to your cluster from your local machine.

```shell
kubectl get nodes
```

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
