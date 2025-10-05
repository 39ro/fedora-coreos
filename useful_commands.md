# Fedora CoreOS Installation
sudo coreos-installer install /dev/sda --ignition-url http://192.168.0.104:8000/coreos.ign --insecure-ignition

# Reset Kubernetes Cluster
sudo kubeadm reset -f

# Journaling and Logging
sudo journalctl -u kubeadm-init.service -f
sudo journalctl -u kubelet -f
sudo journalctl -u kubelet | grep -i "failed\|error"

# Systemd Service Management
sudo systemctl status crio.service
sudo systemctl status kubelet.service
sudo systemctl --failed

sudo systemctl start crio.service
sudo systemctl enable kubelet.service
sudo systemctl start kubelet.service

# Kubernetes Cluster Initialization
sudo kubeadm init --config /etc/kubernetes/kubeadm-config.yaml

# Kubeconfig Setup
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Kubectl Commands
kubectl get pods -A
kubectl get nodes
kubectl describe pod kube-flannel-ds-tldhz -n kube-flannel
kubectl describe node k8s-node

# SELinux Audit
# Check the system audit log to see if SELinux has explicitly logged a denial
sudo ausearch -m avc -ts recent

# Configuration Files
ls /etc/kubernetes/
sudo cat /var/lib/kubelet/config.yaml