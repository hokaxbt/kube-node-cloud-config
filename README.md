# Kubernetes Node Cloud Config

This repository contains my personal [Cloud-init](https://cloudinit.readthedocs.io/en/latest/index.html) configuration to set up a Kubernetes cluster using [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

## Usage

Below is a guide on how to use this cloud-config to set up a Highly Available Kubernetes cluster (3 control-plane nodes) on Hetzner Cloud. Ensure you have [hcloud](https://github.com/hetznercloud/cli) installed.

> [!TIP]
> Kickstart your Kubernetes journey with €20 free credits on Hetzner Cloud! It’s my go-to for reliable, high-performance hosting. Try it yourself with my [referral link](https://hetzner.cloud/?ref=6DFc5bGBrInY).

### Step 1: Create a Private Network & Load Balancer

Create a private network for node communication:

```shell
hcloud network create --name hoka-cluster-network --ip-range 10.0.0.0/16
hcloud network add-subnet hoka-cluster-network --network-zone eu-central --type server --ip-range 10.0.1.0/24
```

Descriptions:

- `--name` specifies the private network name.
- `--ip-range` defines the network CIDR (e.g., 10.0.0.0/16 assigns IPs like 10.0.0.1 to nodes).
- `--network-zone` selects the network zone (e.g., eu-central).

### Step 2: Create a Placement Group

To enhance availability, create a "spread" placement group:

```shell
hcloud placement-group create --name hoka-cluster-group --type spread
```

Descriptions:

- `--name` defines the group name.
- `--type` sets the type; spread ensures VMs are spread across hosts.

### Step 3: Create Control Plane Servers

For control-plane nodes, we’ll use the most affordable ARM machines available on Hetzner. Use the following commands to create three servers:

```shell
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-control-plane-fsn1 --image debian-12 --ssh-key ssh-name --network hoka-cluster-network --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=control-plane
hcloud server create --datacenter hel1-dc2 --type cax11 --name hoka-control-plane-hel1 --image debian-12 --ssh-key ssh-name --network hoka-cluster-network  --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=control-plane
hcloud server create --datacenter nbg1-dc3 --type cax11 --name hoka-control-plane-nbg1 --image debian-12 --ssh-key ssh-name --network hoka-cluster-network  --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=control-plane
```

Descriptions:

- `--datacenter` specifies the datacenter. Use `hcloud datacenter list` to view available options.
- `--type` sets the server type. Check available server types with `hcloud server-type list`.
- `--name` sets the server name, which you can adjust for each control-plane node.
- `--image` specifies the OS image to use, in this case, `debian-12`.
- `--ssh-key` specifies the SSH key to use for access. You can specify a single key with `--ssh-key name` or multiple keys with `--ssh-key name1,name2`. List SSH keys with `hcloud ssh-key list`. Create new SSH Keys with `hcloud ssh-key create`.
- `--network` assigns the private network created in Step 1. Note that control-plane nodes must be in the same zone (e.g., eu-central).
- `--placement-group` ensures nodes in the same datacenter are not hosted on the same host, enhancing availability.

### Step 4: Create Load Balancer

Set up a load balancer:

```shell
hcloud load-balancer create --type lb11 --algorithm-type round_robin --location fsn1 --name hoka-cluster-control-plane-lb
```

Descriptions:

- `--type` sets the load balancer type. Check available types with `hcloud load-balancer-type list`.
- `--algorithm-type` specifies the load-balancing algorithm; `round_robin` distributes traffic evenly.
- `--location` selects the load balancer’s location (use `hcloud location list` to see options).
- `--name` assigns the load balancer’s name.

Add a service for port 6443:

```shell
hcloud load-balancer add-service hoka-cluster-control-plane-lb --protocol tcp --listen-port 6443 --destination-port 6443
```

Descriptions:

- `--protocol` sets the protocol; `tcp` is required for Kubernetes API server communication.
- `--listen-port` specifies the load balancer’s external port (`6443` for Kubernetes).
- `--destination-port` sets the destination port on the nodes (also `6443` for Kubernetes).

Attach load balancer to private network using the following command:

```shell
hcloud load-balancer attach-to-network hoka-cluster-control-plane-lb --network hoka-cluster-network
```

Descriptions:

- `--network` assigns the load balancer to the private network created in Step 1, enabling private communication with nodes.

### Step 5: Initialize the Cluster

Add the first control-plane server as load balancer target:

```shell
hcloud load-balancer add-target hoka-cluster-control-plane-lb --server hoka-control-plane-1 --use-private-ip
```

Descriptions:

- `--server` specifies the server to add as a target (here, `hoka-control-plane-1`).
- `--use-private-ip` directs the load balancer to communicate over the server’s private IP rather than its public IP.

Point your domain to the Load Balancer’s public IP: `YOUR_DOMAIN`.

SSH into `hoka-control-plane-1` and initialize the cluster with kubeadm:

```shell
hcloud server ssh hoka-control-plane-1
```

Initialize the cluster:

```shell
kubeadm init \
	--control-plane-endpoint "YOUR_DOMAIN:6443" \
	--upload-certs \
	--apiserver-advertise-address <hoka-control-plane-1-private-ip> \
	--pod-network-cidr 192.168.0.0/16
```

Descriptions:

- `--control-plane-endpoint` specifies the endpoint for other nodes to join the cluster. Here, it points to your load balancer’s domain and port (e.g., YOUR_DOMAIN:6443).
- `--upload-certs` uploads control-plane certificates to the kubeadm-certs Secret to allow additional control-plane nodes to join securely.
- `--apiserver-advertise-address` sets the IP address for the API server on this node. This is required in order to allow node to communicate via private network.
- `--pod-network-cidr` defines the default CIDR range for the pod network. Here, `192.168.0.0/16` is compatible with Calico.

You should see output similar to:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join <redacted>:6443 --token <redacted> \
	--discovery-token-ca-cert-hash sha256:<redacted> \
	--control-plane --certificate-key <redacted>

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <redacted>:6443 --token <redacted> \
	--discovery-token-ca-cert-hash sha256:<redacted>
```

### Step 6: Add Additional Control Plane Nodes

SSH into `hoka-control-plane-2`:

```shell
hcloud server ssh hoka-control-plane-2
```

Join it to the cluster as a control plane node:

```shell
kubeadm join <redacted>:6443 --token <redacted> \
	--discovery-token-ca-cert-hash sha256:<redacted> \
	--control-plane --certificate-key <redacted>
	--apiserver-advertise-address <hoka-control-plane-2-private-ip>
```

Descriptions:

- `--token` is a unique token required to join the cluster.
- `--discovery-token-ca-cert-hash` is a hash used to validate the CA certificate.
- `--control-plane` specifies that this node will act as a control plane node.
- `--certificate-key` authorizes access to cluster-sensitive data.
- `--apiserver-advertise-address` advertises the private IP address for the API server on this node.

Add the second control-plane server as load balancer target:

```shell
hcloud load-balancer add-target hoka-cluster-control-plane-lb --server hoka-control-plane-2 --use-private-ip
```

Repeat this process for `hoka-control-plane-3`.

### Step 7: Access the Cluster

To access the cluster from your local machine, set up the kubeconfig:

```shell
mkdir -p $HOME/.kube
scp root@<hoka-control-plane-1-ip>:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Descriptions:

- `mkdir -p $HOME/.kube` creates a .kube directory in the home directory.
- `scp` securely copies the kubeconfig file from the server to your local machine.
- `chown $(id -u):$(id -g) $HOME/.kube/config` changes ownership of the kubeconfig file to the current user.

Verify the nodes:

```shell
kubectl get nodes -o wide
```

Example output:

```
NAME                   STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
hoka-control-plane-1   Ready    control-plane   2m58s   v1.31.2   <none>           <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-26-arm64   cri-o://1.31.2
hoka-control-plane-2   Ready    control-plane   87s     v1.31.2   <none>           <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-26-arm64   cri-o://1.31.2
hoka-control-plane-3   Ready    control-plane   48s     v1.31.2   <none>           <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-25-arm64   cri-o://1.31.2
```

You should see that `INTERNAL-IP` and `EXTERNAL-IP` is empty. We will fix this once we installed [Hetzner CCM](https://github.com/hetznercloud/hcloud-cloud-controller-manager).

### Step 8: Install a CNI Network Plugin

Install [Calico](https://www.tigera.io/project-calico/):

```shell
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml
```

Check the Calico pods:

```shell
kubectl get pods -n calico-system
```

Make sure all pods are running except the `calico-kube-controllers`.

### Step 9: Install Hetzner Cloud Controller Manager (CCM)

Create new API key for the project. Then run the following command to create secret:

```shell
kubectl -n kube-system create secret generic hcloud --from-literal=token=<api-key> --from-literal=network=hoka-cluster-network
```

Download the Hetzner CCM manifest:

```shell
curl -OL https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm-networks.yaml
```

Replace the cluster CIDR using the following command:

```shell
sed -i '' 's/10.244.0.0\/16/192.168.0.0\/16/g' ccm-networks.yaml
```

Run the following command to install the Hetzner CCM:

```shell
kubectl apply -f ccm-networks.yaml
```

Verify the setup. First make sure all calico pods are running:

```shell
kubectl get pods -n calico-system
```

Then make sure INTERNAL IP and EXTERNAL IP are updated:

```shell
NAME                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP      OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
hoka-control-plane-1   Ready    control-plane   32m   v1.31.2   10.0.1.1      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-27-arm64   cri-o://1.31.2
hoka-control-plane-2   Ready    control-plane   27m   v1.31.2   10.0.1.2      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-26-arm64   cri-o://1.31.2
hoka-control-plane-3   Ready    control-plane   26m   v1.31.2   10.0.1.3      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-25-arm64   cri-o://1.31.2
```

### Step 10: Add Worker Nodes

Create additional servers for worker nodes:

```shell
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-worker-fsn1-cax11-1 --image debian-12 --ssh-key name1,name2 --network hoka-cluster-network --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=worker
hcloud server create --datacenter hel1-dc2 --type cax11 --name hoka-worker-hel1-cax11-1 --image debian-12 --ssh-key name1,name2 --network hoka-cluster-network --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=worker
hcloud server create --datacenter nbg1-dc3 --type cax11 --name hoka-worker-nbg1-cax11-1 --image debian-12 --ssh-key name1,name2 --network hoka-cluster-network --placement-group hoka-cluster-group --user-data-from-file ./cloud-config --label kubernetes_node_type=worker
```

SSH into each worker node and join it to the cluster:

```shell
kubeadm join <redacted>:6443 --token <redacted> \
	--discovery-token-ca-cert-hash sha256:<redacted> \
	--apiserver-advertise-address <hoka-worker-private-ip>
```

Verify the cluster setup:

```shell
kubectl get nodes -o wide
```

You should see something like the following:

```
NAME                       STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP      OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
hoka-control-plane-1       Ready    control-plane   41m    v1.31.2   10.0.1.1      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-27-arm64   cri-o://1.31.2
hoka-control-plane-2       Ready    control-plane   36m    v1.31.2   10.0.1.2      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-26-arm64   cri-o://1.31.2
hoka-control-plane-3       Ready    control-plane   34m    v1.31.2   10.0.1.3      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-25-arm64   cri-o://1.31.2
hoka-worker-fsn1-cax11-1   Ready    <none>          109s   v1.31.2   10.0.1.5      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-27-arm64   cri-o://1.31.2
hoka-worker-hel1-cax11-1   Ready    <none>          63s    v1.31.2   10.0.1.6      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-26-arm64   cri-o://1.31.2
hoka-worker-nbg1-cax11-1   Ready    <none>          10s    v1.31.2   10.0.1.7      <redacted>       Debian GNU/Linux 12 (bookworm)   6.1.0-25-arm64   cri-o://1.31.2
```

### Step 11: Secure your cluster

We will disable all access via public IP except for SSH, http and https only:

```shell
hcloud firewall create --name kubernetes-node-firewall

hcloud firewall add-rule --direction in --source-ips 0.0.0.0/0 --source-ips ::/0 --protocol tcp --port 22 --description "Allow SSH" kubernetes-node-firewall
hcloud firewall add-rule --direction in --source-ips 0.0.0.0/0 --source-ips ::/0 --protocol tcp --port 80 --description "Allow HTTP" kubernetes-node-firewall
hcloud firewall add-rule --direction in --source-ips 0.0.0.0/0 --source-ips ::/0 --protocol tcp --port 443 --description "Allow HTTPS" kubernetes-node-firewall

hcloud firewall apply-to-resource --type label_selector --label-selector kubernetes_node_type=control-plane kubernetes-node-firewall
hcloud firewall apply-to-resource --type label_selector --label-selector kubernetes_node_type=worker kubernetes-node-firewall
```

Your setup is complete, and the Kubernetes cluster is now fully operational with a highly available control plane and additional worker nodes. You can begin managing your workloads and utilizing Kubernetes features in this environment.

## Clean up resources

You can remove all resources using the following command:

```sh
# Delete servers
hcloud server delete hoka-control-plane-1 hoka-control-plane-2 hoka-control-plane-3 hoka-worker-fsn1-cax11-1 hoka-worker-hel1-cax11-1 hoka-worker-nbg1-cax11-1

# Delete placement group
hcloud placement-group delete hoka-cluster-group

# Delete load balancer
hcloud load-balancer delete hoka-control-plane-lb

# Delete network
hcloud network delete hoka-cluster-network
```

## Known Issues

### Error: `kubeadm: command not found`

Cloud-init may take some time to complete. Check the logs:

```shell
cat /var/log/cloud-init-output.log
```

## Resources

- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [CRI-O](https://cri-o.io/)
- [Calico Quickstart Guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
