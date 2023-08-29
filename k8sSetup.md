# Production cluster setup

## Control Plane setup with kubeadm

### Before you begin

A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.

2 GB or more of RAM per machine (any less will leave little room for your apps).

2 CPUs or more.

Full network connectivity between all machines in the cluster (public or private network is fine).

Unique hostname, MAC address, and product_uuid for every node.

Certain ports are open on your machines. See [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) for more details.

You MUST disable swap in order for the kubelet to work properly.

For example, sudo swapoff -a will disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd.swap, depending how it was configured on your system.

---

### Installing Containerd

Execute the below mentioned instructions:

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

    # sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    # Apply sysctl params without reboot
    sudo sysctl --system

Verify that the br_netfilter, overlay modules are loaded by running the following commands:

    lsmod | grep br_netfilter
    lsmod | grep overlay

Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:

    sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

Download the containerd-\<VERSION>-\<OS>-\<ARCH>.tar.gz archive from https://github.com/containerd/containerd/releases 

    $ tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
    bin/
    bin/containerd-shim-runc-v2
    bin/containerd-shim
    bin/ctr
    bin/containerd-shim-runc-v1
    bin/containerd
    bin/containerd-stress

### systemd

If you intend to start containerd via systemd, you should also download the containerd.service unit file from https://raw.githubusercontent.com/containerd/containerd/main/containerd.service into /usr/local/lib/systemd/system/containerd.service, and run the following commands:

    $ systemctl daemon-reload
    $ systemctl enable --now containerd

### Installing runc

Download the runc.<ARCH> binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as /usr/local/sbin/runc.

    $ install -m 755 runc.amd64 /usr/local/sbin/runc

### Installing CNI plugins

Download the cni-plugins-<OS>-<ARCH>-<VERSION>.tgz archive from https://github.com/containernetworking/plugins/releases, and extract it under /opt/cni/bin:

    $ mkdir -p /opt/cni/bin
    $ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
    ./
    ./macvlan
    ./static
    ./vlan
    ./portmap
    ./host-local
    ./vrf
    ./bridge
    ./tuning
    ./firewall
    ./host-device
    ./sbr
    ./loopback
    ./dhcp
    ./ptp
    ./ipvlan
    ./bandwidth

### Configuring containerd

Default configurations of containerd, Kubernetes and recent Ubuntu versions have some incompatibilities. These can prevent Kubernetes from starting or cause system containers to randomly restart.

To fix these problems, first replace your containerd config file with the default version. This alters some fields that are incorrectly configured on new installs.

    $ containerd config default | sudo tee /etc/containerd/config.toml

Open the config file in your favorite editor and find the following line:

    SystemdCgroup = false

Change the value to true:

    SystemdCgroup = true

This modification enables systemd cgroup management for containerd. Save the file, then restart the containerd service to apply your changes:

    $ sudo service containerd restart

---

## add worker and master IP to /etc/hosts

Adding the worker and master IP addresses to the /etc/hosts file is a recommended practice for setting up a Kubernetes cluster with kubeadm. It is not required, but it can help to improve the performance and reliability of the cluster.

Here are some of the benefits of adding worker and master IP addresses to the /etc/hosts file:

- Improved performance: When the nodes can communicate with each other directly, it can improve the performance of the cluster. This is because DNS lookups can be slow, and by avoiding them, the nodes can communicate with each other more quickly.
- Improved reliability: By adding the worker and master IP addresses to the /etc/hosts file, you are making it less likely that the nodes will lose track of each other. This is because the nodes will always have the IP addresses of the other nodes in the cluster, even if DNS is unavailable.
- Simplified setup: Adding the worker and master IP addresses to the /etc/hosts file can simplify the setup process for Kubernetes clusters. This is because you will not need to configure DNS.

---

## Installing kubeadm, kubectl and kubelet

To set the DNS in resolve.conf for getting Docker images, you need to add the IP addresses of the DNS servers to the file.

The following is an example of a resolve.conf file that sets shecan DNS server:

nameserver 178.22.122.100
nameserver 185.51.200.2

for setting Permanent DNS Nameservers click [here](https://www.tecmint.com/set-permanent-dns-nameservers-in-ubuntu-debian/)

<br>

### These instructions are for Kubernetes 1.28.

Update the apt package index and install packages needed to use the Kubernetes apt repository:

    sudo apt-get update
    
    sudo apt-get install -y apt-transport-https ca-certificates curl

Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

Add the appropriate Kubernetes apt repository:

    # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

## Initializing your control-plane node 

The control-plane node is the machine where the control plane components run, including etcd (the cluster database) and the API Server (which the kubectl command line tool communicates with).

- (Recommended) If you have plans to upgrade this single control-plane kubeadm cluster to high availability you should specify the --control-plane-endpoint to set the shared endpoint for all control-plane nodes. Such an endpoint can be either a DNS name or an IP address of a load-balancer.
- Choose a Pod network add-on, and verify whether it requires any arguments to be passed to kubeadm init. Depending on which third-party provider you choose, you might need to set the --pod-network-cidr to a provider-specific value. See Installing a Pod network add-on.
- (Optional) kubeadm tries to detect the container runtime by using a list of well known endpoints. To use different container runtime or if there are more than one installed on the provisioned node, specify the --cri-socket argument to kubeadm. See Installing a runtime.
- (Optional) Unless otherwise specified, kubeadm uses the network interface associated with the default gateway to set the advertise address for this particular control-plane node's API server. To use a different network interface, specify the --apiserver-advertise-address=<ip-address> argument to kubeadm init. To deploy an IPv6 Kubernetes cluster using IPv6 addressing, you must specify an IPv6 address, for example --apiserver-advertise-address=2001:db8::101

To initialize the control-plane node you can run:

    sudo kubeadm init --apiserver-advertise-address=94.101.184.59 --pod-network-cidr=192.168.0.0/16 -v=7


- --apiserver-advertise-address: The IP address that the API server will advertise to other nodes in the cluster.
- --pod-network-cidr: The CIDR range that will be used for pod networking.
- -v=7: The verbosity level. The higher the value, the more verbose the output.

kubeadm init first runs a series of prechecks to ensure that the machine is ready to run Kubernetes. These prechecks expose warnings and exit on errors. kubeadm init then downloads and installs the cluster control plane components. This may take several minutes. After it finishes you should see:

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user. To make kubectl work for your non-root user, run these commands:

    mkdir -p $HOME/.kube

    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Install Calico

- Install the Tigera Calico operator and custom resource definitions.

        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

- Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see the installation reference.

        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

- Confirm that all of the pods are running with the following command.

        watch kubectl get pods -n calico-system

Wait until each pod has the STATUS of Running.

- Remove the taints on the control plane so that you can schedule pods on it.

        kubectl taint nodes --all node-role.kubernetes.io/control-plane-
        kubectl taint nodes --all node-role.kubernetes.io/master-

It should return the following.

    node/<your-hostname> untainted


Confirm that you now have a node in your cluster with the following command.

    kubectl get nodes -o wide

It should return something like the following.

    NAME              STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
    <your-hostname>   Ready    master   52m   v1.12.2   10.128.0.28   <none>        Ubuntu 18.04.1 LTS   4.15.0-1023-gcp   docker://18.6.1

The command taints the master node with the taint node-role.kubernetes.io/master:NoSchedule. This means that pods will not be scheduled on the master node.

    kubectl taint nodes master node-role.kubernetes.io/master:NoSchedule

## Joining your nodes

The nodes are where your workloads (containers and Pods, etc) run. To add new nodes to your cluster do the following for each machine:

    SSH to the machine

    Become root (e.g. sudo su -)

    Install a runtime if needed

    Run the command that was output by kubeadm init. For example:

    kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>

If you do not have the token, you can get it by running the following command on the control-plane node:

    kubeadm token list

The output is similar to this:

| TOKEN | TTL | EXPIRES | USAGES | DESCRIPTION | EXTRA GROUPS |
|---|---|---|---|---|---|
| 8ewj1p.9r9hcjoqgajrj4gi | 23h | 2018-06-12T02:51:28Z | authentication, signing | The default bootstrap token generated by `kubeadm init`. | system:bootstrappers:kubeadm:default-node-token |

By default, tokens expire after 24 hours. If you are joining a node to the cluster after the current token has expired, you can create a new token by running the following command on the control-plane node:

    kubeadm token create

The output is similar to this:

    5didvk.d09sbcov8ph2amjw

## Enable Auto-completion for Kubectl 

- Check if bash-completion is already installed:

        type _init_completion

- If it is not installed, install using apt or yum, depending on which package manager you are using (usually apt for Ubuntu):

        apt-get install bash-completion 

- Set the kubectl completion script source for your shell sessions:

        kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null

- Set an alias for kubectl as k 

        echo 'alias k=kubectl' >>~/.bashrc

- Enable the alias for auto-completion.

        echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

