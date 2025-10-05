
```shell
python3 -m http.server 8000
```

```shell
sudo coreos-installer install /dev/sda --ignition-url=<url_ign_file> --insecure-ignition
```

In editor, convert encoding of generated .ign file into UTF-8 to remove BOM characters


```shell
docker run --interactive --rm --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release --pretty --strict config.yml > coreos.ign; $content = Get-Content -Path coreos.ign -Raw; [System.IO.File]::WriteAllText("coreos.ign", $content, (New-Object System.Text.UTF8Encoding($false)))
```


## K8S

### crio.service
[CRI-O](https://cri-o.io/) is the engine that runs your containers, and the CRI is the standard language it uses to take orders from Kubernetes.

**How we use it:** We install it on the node and tell the `kubelet` to talk to it by pointing to its communication file (the "socket" at /var/run/crio/crio.sock)


### Admin conf file

#### Copy to your local machine

**Inside the VM,** run these two commands

First, copy the file to your home directory:

`sudo cp /etc/kubernetes/admin.conf /home/core/admin.conf`

Second, change the owner of the new copy so you can read it:

`sudo chown core:core /home/core/admin.conf`

**On your local machine**, run the new scp command

`scp core@<VM_IP>:/home/core/admin.conf $HOME/.kube/config`

#### Edit the copied kubeconfig file

The copied file contains an IP address that needs to be updated.

1.Open the kubeconfig-k8s-node file on your local machine in a text editor.
2.Find the server: line. 
It will look something like this:

```yaml
YAMLclusters:
- cluster:
  certificate-authority-data: <REDACTED_DATA>
  server: https://10.0.2.15:6443  # <-- CHANGE TO YOU YOUR VM'S IP ADDRESS
  name: kubernetes3.Change the IP address in the server: field to your VM's actual, reachable IP address.YAMLclusters:
- cluster:
```