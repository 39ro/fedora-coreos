
```shell
python3 -m http.server 8000
```

```shell
sudo coreos-installer install /dev/sda --ignition-url=<url_ign_file> --insecure-ignition
```

In editor, convert encoding of generated .ign file into UTF-8 to remove BOM characters


```shell
docker run --interactive --rm --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release --pretty --strict config.yml > coreos.ign
```


## K8S

### Enable crio.service
CRI-O is the engine that runs your containers, and the CRI is the standard language it uses to take orders from Kubernetes.

**How we use it:** We install it on the node and tell the `kubelet` to talk to it by pointing to its communication file (the "socket" at /var/run/crio/crio.sock)

```shell
sudo systemctl enable crio.service && sudo systemctl start crio.service
```

### Start a cluster

```shell
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

After Running kubeadm init

1. The command will take a few minutes to run. When it finishes, it will print out:
    - Instructions on how to configure `kubectl` for the core user (this involves creating a .kube directory and copying the admin.conf file).
    - A `kubeadm join` command with a token. Save this command! You will need it to join any future worker nodes to the cluster.
2. After the command succeeds, the /etc/kubernetes/admin.conf file will exist.


**The Detailed Explanation (What is this IP for?)**

The `--pod-network-cidr` does not refer to your VM's IP address or your home network's IP address.

Instead, it defines a private, virtual network inside Kubernetes. Every Pod (the wrapper around your application's containers) that gets created in your cluster will be assigned its own unique IP address from this range.

Think of it as a private neighborhood for your applications to live in.The Most Important Rule: Avoid IP Conflicts

The IP range you choose for your Pods must not overlap with any other networks your VM can see. This includes:1.Your VM's Network: The IP address of your k8s-node VM (e.g., 192.168.1.123) should not be inside the Pod network range.2.Your Proxmox Host Network: The IP of your Proxmox server itself.3.Your Home/Office Network (LAN): The network your computer and Proxmox server are on (e.g., 192.168.1.0/24).

The range 192.168.0.0/16 is a massive range that includes all IPs from 192.168.0.1 to 192.168.255.254. Since most home networks use a small slice like 192.168.1.x, choosing the larger 192.168.0.0/16 for Pods is generally very safe because it doesn't conflict.