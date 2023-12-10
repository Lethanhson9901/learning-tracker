# Using kubeadm

[https://github.com/frankisinfotech/k8s-HA-Multi-Master-Node](https://github.com/frankisinfotech/k8s-HA-Multi-Master-Node)

[Multi Master Kubernetes Cluster set up with Kubeadm and HAproxy](https://www.youtube.com/watch?v=6Gwg80eEuQk&t=177s&ab_channel=FrankTeachesDevOps)

- **Write Vagrant file**
    
    [vagrant.zip](./vagrant.zip)
    
- **Access all VMs**
    
    Here we create an SSH key pair for the `vagrant` user who we are logged in as. We will copy the public key of this pair to the other master and both workers to permit us to use password-less SSH (and SCP) go get from `master-1` to these other nodes in the context of the `vagrant` user which exists on all nodes.
    
    Generate Key Pair on `master-1` node
    
    ```
    ssh-keygen
    ```
    
    Leave all settings to default by pressing `ENTER` at any prompt.
    
    Add this key to the local authorized_keys (`master-1`) as in some commands we scp to ourself.
    
    ```
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    ```
    
    Copy the key to the other hosts. For this step please enter `vagrant` where a password is requested.
    
    Change `/etc/hosts` for all nodes to match config: (not best practice - only for manual testing). 
    
    ```bash
    sudo cat <<EOF | sudo tee /etc/hosts
    127.0.0.1       localhost
    
    # The following lines are desirable for IPv6 capable hosts
    ::1     ip6-localhost   ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    ff02::3 ip6-allhosts
    
    192.168.56.41   new-master-1
    192.168.56.42   new-master-2
    192.168.56.43   new-master-3
    192.168.56.51   new-worker-1
    192.168.56.52   new-worker-2
    192.168.56.53   new-worker-3
    192.168.56.60   new-loadbalancer
    127.0.1.1       ubuntu-jammy    ubuntu-jammy
    EOF
    ```
    
    - **Best Practices for DNS in Kubernetes**
        - **Use a DNS Server:** For production environments, it's recommended to use a proper DNS server or service for hostname resolution. This could be an internal DNS server like BIND, or a cloud provider's DNS service.
        - **Dynamic DNS Updates:** In cloud environments, use services that offer dynamic DNS updates as part of their load balancer or networking services.
        - **Kubernetes Integration:** Employ DNS solutions that integrate well with Kubernetes, providing dynamic, automated management of DNS records.
    
    The option `-o StrictHostKeyChecking=no` tells it not to ask if you want to connect to a previously unknown host. Not best practice in the real world, but speeds things up here.
    
    On `new-master-1:`
    
    ```
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-master-2
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-master-3
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-loadbalancer
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-worker-1
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-worker-2
    ssh-copy-id -o StrictHostKeyChecking=no vagrant@new-worker-3
    ```
    
    Type password if needed: `vagrant`
    
- **Set up load balancer node**
    
    
    ```bash
    INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    LOADBALANCER=$(dig +short new-loadbalancer)
    MASTER_1=$(dig +short new-master-1)
    MASTER_2=$(dig +short new-master-2)
    MASTER_3=$(dig +short new-master-3)
    POD_CIDR=10.244.0.0/16
    SERVICE_CIDR=10.96.0.0/16
    ```
    
    ```bash
    echo "INTERNAL_IP: $INTERNAL_IP"
    echo "LOADBALANCER: $LOADBALANCER"
    echo "MASTER_1: $MASTER_1"
    echo "MASTER_2: $MASTER_2"
    echo "MASTER_3: $MASTER_3"
    echo "POD_CIDR: $POD_CIDR"
    echo "SERVICE_CIDR: $SERVICE_CIDR"
    ```
    
    ### Install Haproxy
    
    ```
    sudo apt update
    sudo apt install -y haproxy
    
    ```
    
    ### Configure haproxy
    
    Append the below lines to **/etc/haproxy/haproxy.cfg**
    
    ```
    frontend kubernetes-frontend
        bind *:6443
        mode tcp
        option tcplog
        default_backend kubernetes-backend
    
    backend kubernetes-backend
        mode tcp
        option tcp-check
        balance roundrobin
        server new-master-1 192.168.56.41:6443 check fall 3 rise 2
        server new-master-2 192.168.56.42:6443 check fall 3 rise 2
        server new-master-3 192.168.56.43:6443 check fall 3 rise 2
    
    ```
    
    ### Restart haproxy service
    
    ```bash
    sudo systemctl restart haproxy
    sudo systemctl status haproxy
    ```
    
    - **Optional: Verify HAProxy Configuration**
        1. **Check if HAProxy is listening on the correct port (6443) using a tool like `netstat` or `ss`:**
            
            ```bash
            sudo ss -tuln | grep 6443
            ```
            
        2. **Test Connectivity to HAProxy**
        - Use the **`nc`** (netcat) command to test connectivity to each of the master node IPs through HAProxy:Replace **`<HA_PROXY_IP>`** with the IP address of your HAProxy load balancer. You should see a connection attempt to port 6443. A "connection refused" error at this stage is normal since Kubernetes API server might not be running yet.
            
            ```bash
            nc -v <HA_PROXY_IP> 6443
            ```
            
- **On all kubernetes nodes**
    
    Disable Firewall
    
    ```
    sudo ufw disable
    ```
    
    ### Disable swap
    
    ```
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    sudo swapoff -a
    ```
    
    ### Update sysctl settings for Kubernetes networking
    
    ```
    # Configure persistent loading of modules
    sudo tee /etc/modules-load.d/containerd.conf <<EOF
    overlay
    br_netfilter
    EOF
    
    # Load at runtime
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    # Ensure sysctl params are set
    sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    
    # Reload configs
    sudo sysctl --system
    ```
    
    ### Install docker engine (docker-ce & containerd)
    
    ```bash
    # Add Docker repo
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    
    # Install containerd
    sudo apt update
    sudo apt install -y docker-ce containerd.io
    
    # Configure containerd and start service
    sudo su -
    ```
    
    ```bash
    mkdir -p /etc/containerd
    containerd config default>/etc/containerd/config.toml
    exit
    ```
    
    ```bash
    # restart containerd
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    systemctl status containerd
    ```
    
    ## Installing kubeadm kubectl kubelet
    
    Copy paste the below snippet one by one in your CLI terminal - This is for both Master and Worker Nodes
    
    ```bash
    sudo apt-get update
    # apt-transport-https may be a dummy package; if so, you can skip that package
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    ```
    
    ```bash
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```
    
    ```bash
    # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
    
    ```bash
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
    
    **Or using old (but stable, compatible version)**
    
    Add Apt repository
    
    ```
    {
      sudo su
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
      echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    }
    
    ```
    
    Install Kubernetes components
    
    ```bash
    apt update && apt install -y kubeadm=1.19.2-00 kubelet=1.19.2-00 kubectl=1.19.2-00
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
    
- **Initialize kubeadm (Only for Master Node)**
    
    Setting up a Kubernetes cluster with stacked etcd and a multi-master configuration involves several detailed steps. Given your infrastructure - 3 master nodes, 3 worker nodes, and 1 load balancer - here's a general outline of how you can proceed from step 4:
    
    ### 1. Initialize the First Master Node:
    
    [https://github.com/kubernetes/kubeadm/issues/2699](https://github.com/kubernetes/kubeadm/issues/2699)
    
    On your first master node, you'll initialize the cluster with `kubeadm`. You need to specify the load balancer endpoint in this command. The load balancer should distribute traffic to all master nodes.
    
    ```bash
    # restart containerd
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    systemctl status containerd
    ```
    
    ```bash
    sudo su
     
    MASTER_1=$(dig +short new-master-1)
    POD_CIDR="192.168.0.0/16"
    
    kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint "new-loadbalancer:6443" --upload-certs --apiserver-advertise-address="$MASTER_1" --pod-network-cidr="$POD_CIDR"  --v=5
    
    #wait for a minute then run
    kubeadm init phase addon kube-proxy --control-plane-endpoint "new-loadbalancer:6443" --pod-network-cidr="$POD_CIDR"  --v=5
    ```
    
    In this command:
    
    - **`-skip-phases=addon/kube-proxy`**: This skips the installation of kube-proxy. This is useful if you plan to use a different networking solution that doesn't require kube-proxy.
    - **`-control-plane-endpoint "new-loadbalancer:6443"`**: This sets the endpoint for the control plane, which is useful for setting up high-availability clusters.
    - **`-upload-certs`**: This is used for securely sharing certificates when setting up additional control plane nodes.
    - **`-apiserver-advertise-address="$MASTER_1"`**: This sets the IP address that the API server advertises to members of the cluster. Here, it uses the IP address resolved from **`new-master-1`**.
    - **`-pod-network-cidr="$POD_CIDR"`**: This specifies the range of IP addresses for the pod network. In this case, it's set to the value of **`POD_CIDR`** which is **`10.244.0.0/16`**.
    - **`-v=5`**: This sets the log level to 5 for more verbose logging, which can be helpful for troubleshooting.
    
    This command will provide you with a `kubeadm join` command for other master nodes and worker nodes. Make sure to note down the token and certificate key. Output is something like that:
    
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
    
      kubeadm join new-loadbalancer:6443 --token jw5kya.03iw0donk27k1095 \
    	--discovery-token-ca-cert-hash sha256:4e06b12476a7d52147f4e5ebdfee7dba254247bf715b594f327c1b5589ca0e9f \
    	--control-plane --certificate-key 586347b349a565f65832444aa4a4a8e2c6122bd5a6f03947397d6728dd2212db
    
    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
    "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join new-loadbalancer:6443 --token jw5kya.03iw0donk27k1095 \
    	--discovery-token-ca-cert-hash sha256:4e06b12476a7d52147f4e5ebdfee7dba254247bf715b594f327c1b5589ca0e9f
    ```
    
    ### 2. Set Up kubeconfig:
    
    After initializing the first master node, set up kubeconfig.
    
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config  
    
    ```
    
    Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the `kube-system` namespace.
    
    ```bash
    kubectl get po -n kube-system
    ```
    
    You verify all the cluster component health statuses using the following command.
    
    ```
    kubectl get --raw='/readyz?verbose'
    ```
    
    You can get the cluster info using the following command.
    
    ```
    kubectl cluster-info dump
    ```
    
    By default, apps won’t get scheduled on the master node. If you **want to use the master node for scheduling apps**, taint the master node. However, this is typically not recommended for production environments due to potential impacts on cluster stability and performance.
    
    ```
    # kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```
    
- **Install Network Plugin:**
    
    Various networking solutions are available for Kubernetes, each with its own features and best practices. The right choice depends on your specific requirements, like network policies, performance needs, and ease of management. Here are some widely-used options:
    
    1. **Calico**:
        - **Best for**: Advanced network policy management, non-overlay and overlay networking.
        - **Features**: Offers fine-grained network policies and is highly scalable. Calico supports both BGP (non-overlay) and VXLAN or IPIP (overlay) networking.
        - **Best Practices**: Use Calico for environments where network security is a priority. Calico’s network policy enforcement can be used with other CNI plugins.
    2. **Weave Net**:
        - **Best for**: Cross-cloud and on-premises deployments.
        - **Features**: Provides a resilient and simple to set up network. It supports encryption, multicast, and dynamic topology.
        - **Best Practices**: Use Weave if you need a robust solution that works well in various environments, including cross-cloud scenarios.
    
    On the first master node, install your chosen network plugin. If you're using Calico, you can run:
    
    [Install Calico networking and network policy for on-premises deployments | Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
    
    ```bash
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml
    
    curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml -O
    
    kubectl create -f custom-resources.yaml
    ```
    
    For short:
    
    ```bash
    kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f [https://docs.projectcalico.org/v3.15/manifests/calico.yaml](https://docs.projectcalico.org/v3.15/manifests/calico.yaml)
    ```
    
    After a couple of minutes, if you check the pods in `kube-system` namespace, you will see calico pods and running CoreDNS pods.
    
    ```bash
    kubectl get po -n kube-system
    ```
    
- 3. Join Other Master Nodes:
    
    For the other master nodes, use the `kubeadm join` command provided by the init output. It should look something like this:
    
    ```bash
    #kubeadm join LOAD_BALANCER_DNS:LOAD_BALANCER_PORT --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key> --apiserver-advertise-address <master-1 ip>
    ```
    
    In my case:
    
    ```bash
    kubeadm join new-loadbalancer:6443 --token l1ves6.yxovm7wplwn0xtbo \
    	--discovery-token-ca-cert-hash sha256:e3459973524aa4cb0424dffd74d1f9ca28e57cd4f8469bc9b20c22d572215801 \
    	--control-plane --certificate-key 3538b002cc8f46b35a88cacbac1243a78a6c94f44c2edc6da5817f6560b18692 --apiserver-advertise-address "$MASTER_1"
    ```
    
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config  
    
    ```
    
    Repeat this on each of the remaining master nodes.
    
    If you missed copying the join command, execute the following command in the master node to recreate the token with the join command.
    
    ```
    kubeadm token create --print-join-command
    ```
    
    **Get the Discovery Token CA Cert Hash**:
    When joining a node to the cluster, you'll also need the discovery token CA cert hash. This can be obtained with the following command:
    
    ```bash
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //’
    ```
    
    ### 4. Join Worker Nodes:
    
    Use the `kubeadm join` command for worker nodes, similar to the one used for joining master nodes but without the `--control-plane` and `--certificate-key` flags.
    
    ```bash
    #kubeadm join LOAD_BALANCER_DNS:LOAD_BALANCER_PORT --token <token> --discovery-token-ca-cert-hash sha256:<hash>
    
    kubeadm join new-loadbalancer:6443 --token jw5kya.03iw0donk27k1095 \
    	--discovery-token-ca-cert-hash sha256:4e06b12476a7d52147f4e5ebdfee7dba254247bf715b594f327c1b5589ca0e9f
    ```
    
    **Verifying the cluster**
    
    ```
    kubectl cluster-info
    kubectl get nodes
    kubectl get cs
    
    ```
    
    Have Fun!!