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