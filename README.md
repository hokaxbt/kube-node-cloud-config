# Kubernetes Node Cloud Init

This repository contains my personal [Cloud-init](https://cloudinit.readthedocs.io/en/latest/index.html) configuration to set up a Kubernetes cluster using [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

## Usage

Below is a guide on how to use this cloud-config to set up a Highly Available Kubernetes cluster (3 control-plane nodes) on Hetzner Cloud. Ensure you have [hcloud](https://github.com/hetznercloud/cli) installed.

> [!TIP]
> Kickstart your Kubernetes journey with €20 free credits on Hetzner Cloud! It’s my go-to for reliable, high-performance hosting. Try it yourself with my [referral link](https://hetzner.cloud/?ref=6DFc5bGBrInY).

Here are a few useful `hcloud` commands:

- `hcloud datacenter list`: List available datacenters
- `hcloud server-type list`: List available server types
- `hcloud ssh-key list`: List SSH keys

### Step 1: Create a Private Network

Create a private network so all nodes can communicate with each other. This network will also be used for the load balancer.

```shell
hcloud network create --name hoka-cluster --ip-range 10.0.0.0/16
hcloud network add-subnet hoka-cluster --network-zone eu-central --type server --ip-range 10.0.0.0/24
```

### Step 2: Create a Placement Group

Next, create a "spread" Placement Group to ensure your VMs are distributed across different hosts. This setup enhances reliability in case one host fails.

```shell
hcloud placement-group create --name hoka-cluster --type spread
```

### Step 3: Create Control Plane Servers

For control-plane nodes, we’ll use the most affordable ARM machines available on Hetzner. Use the following commands to create three servers:

```shell
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-control-plane-1 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-control-plane-2 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-control-plane-3 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
```

### Step 4: Set Up a Control Plane Load Balancer

Run the following command to create a load balancer for the control plane:

```shell
hcloud load-balancer create --type lb11 --algorithm-type round_robin --location fsn1 --name hoka-control-plane-lb
```

Add a service for port 6443:

```shell
hcloud load-balancer add-service hoka-control-plane-lb --protocol tcp --listen-port 6443 --destination-port 6443
```

Add the control-plane nodes as targets:

```shell
hcloud load-balancer add-target hoka-control-plane-lb --server hoka-control-plane-1 --use-private-ip
hcloud load-balancer add-target hoka-control-plane-lb --server hoka-control-plane-2 --use-private-ip
hcloud load-balancer add-target hoka-control-plane-lb --server hoka-control-plane-3 --use-private-ip
```

Point your domain to the Load Balancer’s public IP: `LOAD_BALANCER_DNS`.

### Step 5: Initialize the Cluster

SSH into `hoka-control-plane-1`:

```shell
hcloud server ssh hoka-control-plane-1
```

Initialize the cluster:

```shell
kubeadm init \
	--control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
	--upload-certs \
	--pod-network-cidr 192.168.0.0/16
```

You should see output similar to the following:

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
```

Repeat this for `hoka-control-plane-3`.

### Step 7: Access the Cluster

To access the cluster from your local machine, set up the kubeconfig:

```shell
mkdir -p $HOME/.kube
scp root@<hoka-control-plane-1-ip>:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the nodes:

```shell
kubectl get nodes
```

```
NAME                   STATUS   ROLES           AGE     VERSION
hoka-control-plane-1   Ready    control-plane   12m     v1.31.2
hoka-control-plane-2   Ready    control-plane   6m52s   v1.31.2
hoka-control-plane-3   Ready    control-plane   5m17s   v1.31.2
```

### Step 8: Install a CNI Network Plugin

Install [Calico](https://www.tigera.io/project-calico/) as the CNI:

```shell
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml
```

Check the status:

```shell
kubectl get pods -n calico-system
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7dd59485cf-88txb   1/1     Running   0          41s
calico-node-4kh2v                          1/1     Running   0          41s
calico-node-jj6cd                          0/1     Running   0          41s
calico-node-qzjfr                          1/1     Running   0          41s
calico-typha-6d76554cb8-r7ss4              1/1     Running   0          41s
calico-typha-6d76554cb8-svfmz              1/1     Running   0          33s
csi-node-driver-lb47p                      2/2     Running   0          41s
csi-node-driver-lbcfl                      2/2     Running   0          41s
csi-node-driver-p42wv                      2/2     Running   0          41s
```

### Step 9: Add Worker Nodes

Create additional servers for worker nodes:

```shell
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-worker-2cpu-4gb-40gb-1 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-worker-2cpu-4gb-40gb-2 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
hcloud server create --datacenter fsn1-dc14 --type cax11 --name hoka-worker-2cpu-4gb-40gb-3 --image debian-12 --ssh-key name1,name2 --network hoka-cluster --placement-group hoka-cluster --user-data-from-file ./cloud-config
```

SSH into each worker node and join it to the cluster:

```shell
kubeadm join <redacted>:6443 --token <redacted> \
	--discovery-token-ca-cert-hash sha256:<redacted>
```

Verify the cluster setup:

```shell
kubectl get nodes
```

```
NAME                          STATUS   ROLES           AGE   VERSION
hoka-control-plane-1          Ready    control-plane   39m   v1.31.2
hoka-control-plane-2          Ready    control-plane   33m   v1.31.2
hoka-control-plane-3          Ready    control-plane   32m   v1.31.2
hoka-worker-2cpu-4gb-40gb-1   Ready    <none>          78s   v1.31.2
hoka-worker-2cpu-4gb-40gb-2   Ready    <none>          57s   v1.31.2
hoka-worker-2cpu-4gb-40gb-3   Ready    <none>          11s   v1.31.2
```

Your setup is complete, and the Kubernetes cluster is now fully operational with a highly available control plane and additional worker nodes. You can begin managing your workloads and utilizing Kubernetes features in this environment.

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
