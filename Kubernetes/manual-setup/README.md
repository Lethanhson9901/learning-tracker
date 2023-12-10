# Manual Setup

[https://github.com/mmumshad/kubernetes-the-hard-way](https://github.com/mmumshad/kubernetes-the-hard-way)

[vagrant.zip](./Kubernetes/manual-setup/vagrant.zip)

![Untitled](./Kubernetes/manual-setup/Untitled.png)

- 04: **Provisioning a CA and Generating TLS Certificates**
    
    You're setting up a Public Key Infrastructure (PKI) using OpenSSL for a Kubernetes cluster. This process involves creating a Certificate Authority (CA), and then generating TLS certificates for various Kubernetes components.
    
    - **Certificate Authority**
        - **What it is**: The CA is like a trusted entity that issues digital certificates. Think of it as a passport office that gives out passports (certificates).
        - **What you do**: You create a private key for the CA and then use it to create a CA certificate. This CA certificate will later be used to "sign" (approve) other certificates.
        
        Before setting up the CA, we need to determine the IP addresses of the hosts (machines) in our cluster. These addresses will be used as "Subject Alternative Names" (SANs) in the certificates. The information will be retrieved from the **`/etc/hosts`** file on each machine.
        
        - **What it is**: SANs are like additional identities for a server. Just like a person can have multiple IDs (passport, driver's license), a server can have multiple names (IP addresses, DNS names).
        - **What you do**: You gather IP addresses of your Kubernetes nodes. These IPs will be included in the SANs of your certificates, allowing the servers to be identified by these IPs.
        1. **Set Up Environment Variables**:
        - We create some variables to store the IP addresses of key hosts in our Kubernetes cluster.
        
        ```bash
        MASTER_1=$(dig +short master-1)
        MASTER_2=$(dig +short master-2)
        MASTER_3=$(dig +short master-3)
        LOADBALANCER=$(dig +short loadbalancer)
        SERVICE_CIDR=10.96.0.0/24
        API_SERVICE=$(echo $SERVICE_CIDR | awk 'BEGIN {FS="."} ; { printf("%s.%s.%s.1", $1, $2, $3) }')
        ```
        
        We also define a variable for the Kubernetes service CIDR range (**`SERVICE_CIDR`**) and compute the API server service address (**`API_SERVICE`**) within that CIDR range. This address is always **`.1`**.
        
        ```bash
        echo $MASTER_1
        echo $MASTER_2
        echo $MASTER_3
        echo $LOADBALANCER
        echo $SERVICE_CIDR
        echo $API_SERVICE
        ```
        
        ```bash
        # Create private key for CA
        openssl genrsa -out ca.key 2048
        
          # Create CSR using the private key
        openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA/O=Kubernetes" -out ca.csr
        
          # Self sign the csr using its own private key
        openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
        ```
        
        The ca.crt is the Kubernetes Certificate Authority certificate and ca.key is the Kubernetes Certificate Authority private key. You will use the ca.crt file in many places, so it will be copied to many places. The ca.key is used by the CA for signing certificates. And it should be securely stored. In this case our master node(s) is our CA server as well, so we will store it on master node(s). There is no need to copy this file elsewhere.
        
    - **Client and Server Certificates**
        
        Now, you generate certificates for different components of Kubernetes. Each component (like kube-apiserver, kube-controller-manager, etc.) needs its own certificate for secure communication.
        
        **Generating the Admin Client Certificate:**
        
        - **What it is**: This is like a special ID for the Kubernetes admin to access and manage the cluster.
        - **What you do**: You create a private key and a certificate for the admin user. This certificate is signed by your CA.
        
        Here's how to generate it:
        
        ```bash
        # Generate a private key for the admin user (2048 bits is a common key length)
        openssl genrsa -out admin.key 2048
        
        # Generate a Certificate Signing Request (CSR) for the admin user
        openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
        
        # Sign the certificate for the admin user using the CA server's private key
        openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 1000
        
        ```
        
        Explanation of the commands:
        
        1. `openssl genrsa -out admin.key 2048`: This generates a private key for the admin user with a key length of 2048 bits.
        2. `openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr`: This creates a Certificate Signing Request (CSR) for the admin user. The subject (`subj`) includes the Common Name (CN) as "admin" and specifies that the user is part of the "system:masters" group, which grants administrative privileges.
        3. `openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 1000`: This signs the CSR with the CA certificate (`ca.crt`) and its private key (`ca.key`). It creates the admin client certificate (`admin.crt`) with a validity of 1000 days.
        
        After running these commands, you'll have generated the following files:
        
        - `admin.key`: The private key for the admin user.
        - `admin.crt`: The admin client certificate, which provides administrative access to the Kubernetes cluster.
        
        You can configure `admin.key` and `admin.crt` to be used with the `kubectl` tool to perform administrative functions on the Kubernetes cluster.
        
        **Kubelet Client Certificates**
        
        - **What you do**: You’re skipping this for now but will create them later for each worker node in your cluster.
        
        **Controller Manager, Scheduler, and Proxy Certificates**
        
        - **What they are**: These are certificates for different components that help run the Kubernetes cluster.
        - **What you do**: For each of these components, you generate a private key and a certificate, and get them signed by the CA.
        
        ```bash
        # Setup Controller Manager Client Certificate
        openssl genrsa -out kube-controller-manager.key 2048
        
        openssl req -new -key kube-controller-manager.key \
            -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" -out kube-controller-manager.csr
        
        openssl x509 -req -in kube-controller-manager.csr \
            -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
        
        # Setup Kube Proxy Client Certificate
        openssl genrsa -out kube-proxy.key 2048
        
        openssl req -new -key kube-proxy.key \
            -subj "/CN=system:kube-proxy/O=system:node-proxier" -out kube-proxy.csr
        
        openssl x509 -req -in kube-proxy.csr \
            -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000
        
        # Setup Scheduler Client Certificate
        openssl genrsa -out kube-scheduler.key 2048
        
        openssl req -new -key kube-scheduler.key \
            -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" -out kube-scheduler.csr
        
        openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000
        ```
        
        ### Kubernetes API Server Certificate
        
        - **What it is**: This certificate is for the kube-apiserver, which is a crucial part of the Kubernetes control plane.
        - **Why it's special**: This certificate needs to include all possible names (IPs, DNS names) that could be used to reach the API server.
        - **What you do**: You first create a configuration file (**`openssl.cnf`**) to specify all these names. Then, you generate a private key and a certificate for the kube-apiserver using this config file.
        
        ```bash
        cat > openssl.cnf <<EOF
        [req]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]
        [v3_req]
        basicConstraints = critical, CA:FALSE
        keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
        extendedKeyUsage = serverAuth
        subjectAltName = @alt_names
        [alt_names]
        DNS.1 = kubernetes
        DNS.2 = kubernetes.default
        DNS.3 = kubernetes.default.svc
        DNS.4 = kubernetes.default.svc2
        DNS.5 = kubernetes.default.svc.cluster
        DNS.6 = kubernetes.default.svc.cluster.local
        IP.1 = ${API_SERVICE}
        IP.2 = ${MASTER_1}
        IP.3 = ${MASTER_2}
        IP.4 = ${MASTER_3}  
        IP.5 = ${LOADBALANCER}
        IP.6 = 127.0.0.1
        EOF
        ```
        
        ```bash
        openssl genrsa -out kube-apiserver.key 2048
        
        openssl req -new -key kube-apiserver.key \
            -subj "/CN=kube-apiserver/O=Kubernetes" -out kube-apiserver.csr -config openssl.cnf
        
        openssl x509 -req -in kube-apiserver.csr \
          -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
        
        ```
        
        **The Kubelet Client Certificate**
        
        ```bash
        cat > openssl-kubelet.cnf <<EOF
        [req]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]
        [v3_req]
        basicConstraints = critical, CA:FALSE
        keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
        extendedKeyUsage = clientAuth
        EOF
        
        openssl genrsa -out apiserver-kubelet-client.key 2048
        
        openssl req -new -key apiserver-kubelet-client.key \
            -subj "/CN=kube-apiserver-kubelet-client/O=system:masters" -out apiserver-kubelet-client.csr -config openssl-kubelet.cnf
        
        openssl x509 -req -in apiserver-kubelet-client.csr \
          -CA ca.crt -CAkey ca.key -CAcreateserial  -out apiserver-kubelet-client.crt -extensions v3_req -extfile openssl-kubelet.cnf -days 1000
        ```
        
        **The ETCD Server Certificate**
        
        ```bash
        cat > openssl-etcd.cnf <<EOF
        [req]
        req_extensions = v3_req
        distinguished_name = req_distinguished_name
        [req_distinguished_name]
        [ v3_req ]
        basicConstraints = CA:FALSE
        keyUsage = nonRepudiation, digitalSignature, keyEncipherment
        subjectAltName = @alt_names
        [alt_names]
        IP.1 = ${MASTER_1}
        IP.2 = ${MASTER_2}
        IP.3 = ${MASTER_3}
        IP.4 = 127.0.0.1
        EOF
        
        openssl genrsa -out etcd-server.key 2048
        
        openssl req -new -key etcd-server.key \
            -subj "/CN=etcd-server/O=Kubernetes" -out etcd-server.csr -config openssl-etcd.cnf
        
        openssl x509 -req -in etcd-server.csr \
            -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
        ```
        
        **The Service Account Key Pair**
        
        ```bash
        openssl genrsa -out service-account.key 2048
        
         openssl req -new -key service-account.key \
            -subj "/CN=service-accounts/O=Kubernetes" -out service-account.csr
        
        openssl x509 -req -in service-account.csr \
            -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 1000
        ```
        
        ## Verify the PKI
        
        Run the following, and select option 1 to check all required certificates were generated.
        
        ```
        ./cert_verify.sh
        
        ```
        
        > Expected output
        > 
        
        ```
        PKI generated correctly!
        
        ```
        
        If there are any errors, please review above steps and then re-verify
        
        ## Distribute the Certificates
        
        Copy the appropriate certificates and private keys to each instance:
        
        ```bash
        #!/bin/bash
        
        # Copy certificates to master nodes
        for instance in master-1 master-2 master-3; do
          scp -o StrictHostKeyChecking=no ca.crt ca.key kube-apiserver.key kube-apiserver.crt \
            apiserver-kubelet-client.crt apiserver-kubelet-client.key \
            service-account.key service-account.crt \
            etcd-server.key etcd-server.crt \
            kube-controller-manager.key kube-controller-manager.crt \
            kube-scheduler.key kube-scheduler.crt \
            ${instance}:~/
        done
        
        # Copy certificates to worker nodes
        for instance in worker-1 worker-2 worker-3; do
          scp ca.crt kube-proxy.crt kube-proxy.key ${instance}:~/
        done
        ```
        
- 05: **Generating Kubernetes Configuration Files for Authentication**
    
    In this lab, you're generating Kubernetes configuration files (kubeconfigs) which are crucial for Kubernetes clients to connect and authenticate with the Kubernetes API Servers.
    
    **Overview**
    
    - **Kubeconfigs**: Configuration files for Kubernetes clients.
    - **Purpose**: To enable clients to locate and authenticate to the Kubernetes API servers.
    - **Best Practice**: Use file paths to certificates in kubeconfigs for services. Embed certificate data directly in user configs like **`admin.kubeconfig`**.
    
    ### **Generating Client Authentication Configs**
    
    1. **Kubernetes Public IP Address**:
        - The IP of the load balancer is stored in a variable (**`LOADBALANCER`**) to be used in kubeconfigs.
        
        ```bash
        LOADBALANCER=$(dig +short loadbalancer)
        ```
        
        - For services running on the master nodes like controller manager and scheduler, **`127.0.0.1`** (localhost) is used.
    2. **Generating Kubeconfigs**:
        - For each Kubernetes component (kube-proxy, kube-controller-manager, kube-scheduler) and the admin user, you generate a kubeconfig.
        - Each kubeconfig file includes the cluster details, credentials, and context settings.
        - These files are generated using **`kubectl config`** commands.
    
    ### Components:
    
    - **kube-proxy**:
        - The kube-proxy kubeconfig points to the load balancer's IP.
    
    ```bash
    kubectl config set-cluster kubernetes-the-hard-way \
        --certificate-authority=/var/lib/kubernetes/pki/ca.crt \
        --server=https://${LOADBALANCER}:6443 \
        --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config set-credentials system:kube-proxy \
        --client-certificate=/var/lib/kubernetes/pki/kube-proxy.crt \
        --client-key=/var/lib/kubernetes/pki/kube-proxy.key \
        --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config set-context default \
        --cluster=kubernetes-the-hard-way \
        --user=system:kube-proxy \
        --kubeconfig=kube-proxy.kubeconfig
    
    kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
    ```
    
    Result: `kube-proxy.kubeconfig`
    
    ```bash
    vagrant@master-1:~$ cat kube-proxy.kubeconfig 
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /var/lib/kubernetes/pki/ca.crt
        server: https://192.168.56.30:6443
      name: kubernetes-the-hard-way
    contexts:
    - context:
        cluster: kubernetes-the-hard-way
        user: system:kube-proxy
      name: default
    current-context: default
    kind: Config
    preferences: {}
    users:
    - name: system:kube-proxy
      user:
        client-certificate: /var/lib/kubernetes/pki/kube-proxy.crt
        client-key: /var/lib/kubernetes/pki/kube-proxy.key
    ```
    
    - **kube-controller-manager** and **kube-scheduler**:
        - These kubeconfigs point to the localhost API server.
    
    ```bash
    kubectl config set-cluster kubernetes-the-hard-way \
        --certificate-authority=/var/lib/kubernetes/pki/ca.crt \
        --server=https://127.0.0.1:6443 \
        --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config set-credentials system:kube-controller-manager \
        --client-certificate=/var/lib/kubernetes/pki/kube-controller-manager.crt \
        --client-key=/var/lib/kubernetes/pki/kube-controller-manager.key \
        --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config set-context default \
        --cluster=kubernetes-the-hard-way \
        --user=system:kube-controller-manager \
        --kubeconfig=kube-controller-manager.kubeconfig
    
    kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
    ```
    
    Result: `kube-controller-manager.kubeconfig`
    
    Kube Scheduler
    
    ```bash
    kubectl config set-cluster kubernetes-the-
    hard-way \
        --certificate-authority=/var/lib/kubernetes/pki/ca.crt \
        --server=https://127.0.0.1:6443 \
        --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config set-credentials system:kube-scheduler \
        --client-certificate=/var/lib/kubernetes/pki/kube-scheduler.crt \
        --client-key=/var/lib/kubernetes/pki/kube-scheduler.key \
        --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config set-context default \
        --cluster=kubernetes-the-hard-way \
        --user=system:kube-scheduler \
        --kubeconfig=kube-scheduler.kubeconfig
    
    kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
    ```
    
    - **admin**:
        - The admin kubeconfig has the certificate information embedded and also points to the localhost API server.
    
    ```bash
    kubectl config set-cluster kubernetes-the-hard-way \
        --certificate-authority=ca.crt \
        --embed-certs=true \
        --server=https://127.0.0.1:6443 \
        --kubeconfig=admin.kubeconfig
    
    kubectl config set-credentials admin \
        --client-certificate=admin.crt \
        --client-key=admin.key \
        --embed-certs=true \
        --kubeconfig=admin.kubeconfig
    
    kubectl config set-context default \
        --cluster=kubernetes-the-hard-way \
        --user=admin \
        --kubeconfig=admin.kubeconfig
    
    kubectl config use-context default --kubeconfig=admin.kubeconfig
    ```
    
    Result: `admin.kubeconfig`
    
    ### **Distributing the Kubernetes Configuration Files**
    
    - **To Worker Nodes**:
        - The **`kube-proxy.kubeconfig`** is copied to each worker node. This enables kube-proxy on the worker nodes to communicate with the Kubernetes API server.
    - **To Master Nodes**:
        - The **`admin.kubeconfig`**, **`kube-controller-manager.kubeconfig`**, and **`kube-scheduler.kubeconfig`** are copied to each master node.
    
    ```bash
    for instance in worker-1 worker-2 worker-3; do
      scp kube-proxy.kubeconfig ${instance}:~/
    done
    ```
    
    ```bash
    for instance in master-1 master-2 master-3; do
      scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
    done
    ```
    
    **Optional - Check kubeconfigs**
    
    At `master-1` and `master-2` and `master-3` nodes, run the following, selecting option 2
    
    ```bash
    ./cert_verify.sh
    ```
    
- 06: **Generating the Data Encryption Config and Key**
    
    In this lab, you are tasked with generating an encryption key and creating a configuration file for encrypting Kubernetes Secrets. This is a crucial step in securing sensitive data stored in your Kubernetes cluster
    
    **Overview**
    
    - **Goal**: To encrypt cluster data at rest (specifically Kubernetes Secrets) stored within **`etcd`**.
    - **Why it's important**: Encrypting data at rest enhances the security of sensitive information stored in your Kubernetes cluster.
    
    **Steps to Generate Encryption Key and Config File**
    
    1. **Generating the Encryption Key**:
    
    ```bash
    ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
    ```
    
    - An encryption key is generated using **`head`** and **`base64`** commands.
    - This key will be used to encrypt the cluster data.
    
    2. **Creating the Encryption Config File**:
    
    ```bash
    cat > encryption-config.yaml <<EOF
    kind: EncryptionConfig
    apiVersion: v1
    resources:
      - resources:
          - secrets
        providers:
          - aescbc:
              keys:
                - name: key1
                  secret: ${ENCRYPTION_KEY}
          - identity: {}
    EOF
    ```
    
    3. **Distributing the Encryption Config File**:
    
    - You use **`scp`** to copy the **`encryption-config.yaml`** file to each controller instance.
    
    ```bash
    for instance in master-1 master-2 master-3; do
      scp encryption-config.yaml ${instance}:~/
    done
    ```
    
    - Then, you securely move the encryption config file into the **`/var/lib/kubernetes/`** directory on each controller instance.
    
    ```bash
    for instance in master-1 master-2 master-3; do
      ssh ${instance} sudo mkdir -p /var/lib/kubernetes/
      ssh ${instance} sudo mv encryption-config.yaml /var/lib/kubernetes/
    done
    ```
    
    - This step ensures that the encryption configuration is available on all master nodes of the cluster.
    
- 07: **Bootstrapping the etcd Cluster**
    
    In this lab, you are setting up a high-availability, secure etcd cluster for your Kubernetes environment. Etcd is a distributed key-value store that Kubernetes uses to store all its cluster data.
    
    ### **Overview**
    
    - **Goal**: Bootstrap a three-node etcd cluster for Kubernetes.
    - **Why Etcd**: Kubernetes components are stateless and store their state in etcd.
    - **High Availability and Security**: The etcd cluster is configured for high availability and secure remote access.
    
    ### **Steps for Bootstrapping the etcd Cluster**
    
    ### 1. **Download and Install etcd Binaries**:
    
    - Download etcd from the official GitHub repository.
    - Extract and install **`etcd`** and **`etcdctl`** (the command line utility for etcd).
    
    ```bash
    ETCD_VERSION="v3.5.9"
    wget -q --show-progress --https-only --timestamping \
      "https://github.com/coreos/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz"
    
    tar -xvf etcd-${ETCD_VERSION}-linux-amd64.tar.gz
    sudo mv etcd-${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin/
    ```
    
    Or: (Not recommend)
    
    ```bash
    ETCD_VER=v3.5.9
    
    # choo
    se either URL
    GOOGLE_URL=https://storage.googleapis.com/etcd
    GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
    DOWNLOAD_URL=${GOOGLE_URL}
    
    rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
    rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test
    
    curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
    tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
    rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
    
    /tmp/etcd-download-test/etcd --version
    /tmp/etcd-download-test/etcdctl version
    /tmp/etcd-download-test/etcdutl version
    
    sudo mv /tmp/etcd-download-test/etcd* /usr/local/bin/
    ```
    
    ### 2. **Configure the etcd Server**:
    
    - **Secure Certificates**: Copy the **`etcd-server`** key and certificate, and the CA certificate into appropriate directories.
    - **Set Permissions**: Ensure these files are securely accessible only by authorized users.
    
    ```bash
    sudo mkdir -p /etc/etcd /var/lib/etcd /var/lib/kubernetes/pki
    sudo cp etcd-server.key etcd-server.crt /etc/etcd/
    sudo cp ca.crt /var/lib/kubernetes/pki/
    sudo chown root:root /etc/etcd/*
    sudo chmod 600 /etc/etcd/*
    sudo chown root:root /var/lib/kubernetes/pki/*
    sudo chmod 600 /var/lib/kubernetes/pki/*
    sudo ln -s /var/lib/kubernetes/pki/ca.crt /etc/etcd/ca.crt
    ```
    
    - **Internal IP Address**: Determine the internal IP addresses of the master nodes to be used for etcd peer communication.
    - **Unique Names**: Assign a unique name to each etcd member based on the hostname of the instance.
    
    ```bash
    INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    MASTER_1=$(dig +short master-1)
    MASTER_2=$(dig +short master-2)
    MASTER_3=$(dig +short master-3)
    ETCD_NAME=$(hostname -s)
    ```
    
    ```bash
    echo $INTERNAL_IP
    echo $MASTER_1
    echo $MASTER_2
    echo $MASTER_3
    echo $ETCD_NAME
    ```
    
    ### 3. **Create etcd systemd Unit File**:
    
    - Define a **`systemd`** service for etcd, specifying various flags like:
        - etcd member name, certificates, and key files for secure communication.
        - URLs for client and peer communication.
        - Initial etcd cluster configuration, including member URLs.
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/etcd.service
    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos
    
    [Service]
    ExecStart=/usr/local/bin/etcd \\
      --name ${ETCD_NAME} \\
      --cert-file=/etc/etcd/etcd-server.crt \\
      --key-file=/etc/etcd/etcd-server.key \\
      --peer-cert-file=/etc/etcd/etcd-server.crt \\
      --peer-key-file=/etc/etcd/etcd-server.key \\
      --trusted-ca-file=/etc/etcd/ca.crt \\
      --peer-trusted-ca-file=/etc/etcd/ca.crt \\
      --peer-client-cert-auth \\
      --client-cert-auth \\
      --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
      --advertise-client-urls https://${INTERNAL_IP}:2379 \\
      --initial-cluster-token etcd-cluster-0 \\
      --initial-cluster master-1=https://${MASTER_1}:2380,master-2=https://${MASTER_2}:2380,master-3=https://${MASTER_3}:2380 \\
      --initial-cluster-state new \\
      --data-dir=/var/lib/etcd
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    ### 4. **Start the etcd Server**:
    
    - Reload **`systemd`** to recognize the new etcd service.
    - Enable the etcd service to start on boot.
    - Start the etcd service.
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd
    ```
    
    ### **Verification**
    
    - **List etcd Cluster Members**: Use **`etcdctl`** to verify the members of the etcd cluster.
    - The output should list **`master-1`** and **`master-2`** and **`master-3`**, confirming that the etcd cluster has been successfully bootstrapped.
    
    ```bash
    sudo ETCDCTL_API=3 etcdctl member list \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/etcd/ca.crt \
      --cert=/etc/etcd/etcd-server.crt \
      --key=/etc/etcd/etcd-server.key
    ```
    
    ### **Summary**
    
    - You've bootstrapped a three-node etcd cluster, a critical component for Kubernetes state storage.
    - The cluster is configured for high availability, and all communications are secured using TLS certificates.
    - Each step ensures that etcd is installed, configured, and started properly on both master nodes.
    
    ### **Next Steps**
    
    - The next phase involves bootstrapping the Kubernetes control plane, setting up essential components like the API server, controller manager, and scheduler.
    
    ### **Note**
    
    - Running these commands on each controller instance ensures that the etcd cluster is set up across multiple nodes for high availability.
    - Using tools like **`tmux`** can simplify running commands on multiple instances simultaneously.
- 08: **Bootstrapping the Kubernetes Control Plane**
    
    In this lab, you're setting up the Kubernetes control plane across 3 compute instances (`master-1` , `master-2` and `master-3`) and configuring an external load balancer.
    
    ### Overview
    
    - **Goal**: Bootstrap the Kubernetes control plane for high availability.
    - **Components**: Install and configure Kubernetes API Server, Scheduler, and Controller Manager on each node.
    - **Load Balancer**: Set up an external load balancer for the Kubernetes API Servers.
    
    ### Steps to Provision the Kubernetes Control Plane
    
    ### 1. **Download and Install Kubernetes Binaries**:
    
    - Download the latest official Kubernetes binaries (`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubectl`).
    
    ```bash
    KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
    
    wget -q --show-progress --https-only --timestamping \
      "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kube-apiserver" \
      "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kube-controller-manager" \
      "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kube-scheduler" \
      "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl"
    ```
    
    - Install these binaries by moving them to `/usr/local/bin/` and setting the execute permission.
    
    ```bash
    chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
    sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
    ```
    
    ### 2. **Configure the Kubernetes API Server**:
    
    - Place and secure key pairs in the Kubernetes data directory (`/var/lib/kubernetes/pki`).
    - Retrieve internal IP addresses for communication within the cluster and the load balancer's IP for external access.
    - Set CIDR ranges for pods and services within the cluster.
    - Create a systemd unit file for the `kube-apiserver` service with detailed configuration flags.
    
    ```bash
    sudo mkdir -p /var/lib/kubernetes/pki
    
    # Only copy CA keys as we'll need them again for workers.
    sudo cp ca.crt ca.key /var/lib/kubernetes/pki
    for c in kube-apiserver service-account apiserver-kubelet-client etcd-server kube-scheduler kube-controller-manager
    do
      sudo mv "$c.crt" "$c.key" /var/lib/kubernetes/pki/
    done
    sudo chown root:root /var/lib/kubernetes/pki/*
    sudo chmod 600 /var/lib/kubernetes/pki/*
    ```
    
    ```bash
    INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    LOADBALANCER=$(dig +short loadbalancer)
    MASTER_1=$(dig +short master-1)
    MASTER_2=$(dig +short master-2)
    MASTER_3=$(dig +short master-3)
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
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/kubernetes/kubernetes
    
    [Service]
    ExecStart=/usr/local/bin/kube-apiserver \\
      --advertise-address=${INTERNAL_IP} \\
      --allow-privileged=true \\
      --apiserver-count=3 \\
      --audit-log-maxage=30 \\
      --audit-log-maxbackup=3 \\
      --audit-log-maxsize=100 \\
      --audit-log-path=/var/log/audit.log \\
      --authorization-mode=Node,RBAC \\
      --bind-address=0.0.0.0 \\
      --client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
      --enable-admission-plugins=NodeRestriction,ServiceAccount \\
      --enable-bootstrap-token-auth=true \\
      --etcd-cafile=/var/lib/kubernetes/pki/ca.crt \\
      --etcd-certfile=/var/lib/kubernetes/pki/etcd-server.crt \\
      --etcd-keyfile=/var/lib/kubernetes/pki/etcd-server.key \\
      --etcd-servers=https://${MASTER_1}:2379,https://${MASTER_2}:2379,https://${MASTER_3}:2379 \\
      --event-ttl=1h \\
      --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
      --kubelet-certificate-authority=/var/lib/kubernetes/pki/ca.crt \\
      --kubelet-client-certificate=/var/lib/kubernetes/pki/apiserver-kubelet-client.crt \\
      --kubelet-client-key=/var/lib/kubernetes/pki/apiserver-kubelet-client.key \\
      --runtime-config=api/all=true \\
      --service-account-key-file=/var/lib/kubernetes/pki/service-account.crt \\
      --service-account-signing-key-file=/var/lib/kubernetes/pki/service-account.key \\
      --service-account-issuer=https://${LOADBALANCER}:6443 \\
      --service-cluster-ip-range=${SERVICE_CIDR} \\
      --service-node-port-range=30000-32767 \\
      --tls-cert-file=/var/lib/kubernetes/pki/kube-apiserver.crt \\
      --tls-private-key-file=/var/lib/kubernetes/pki/kube-apiserver.key \\
      --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    ### 3. **Configure the Kubernetes Controller Manager**:
    
    - Move the `kube-controller-manager` kubeconfig into place.
    - Create a systemd unit file for the `kube-controller-manager` service.
    
    ```bash
    sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
    ```
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/kubernetes/kubernetes
    
    [Service]
    ExecStart=/usr/local/bin/kube-controller-manager \\
      --allocate-node-cidrs=true \\
      --authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
      --authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
      --bind-address=127.0.0.1 \\
      --client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
      --cluster-cidr=${POD_CIDR} \\
      --cluster-name=kubernetes \\
      --cluster-signing-cert-file=/var/lib/kubernetes/pki/ca.crt \\
      --cluster-signing-key-file=/var/lib/kubernetes/pki/ca.key \\
      --controllers=*,bootstrapsigner,tokencleaner \\
      --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
      --leader-elect=true \\
      --node-cidr-mask-size=24 \\
      --requestheader-client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
      --root-ca-file=/var/lib/kubernetes/pki/ca.crt \\
      --service-account-private-key-file=/var/lib/kubernetes/pki/service-account.key \\
      --service-cluster-ip-range=${SERVICE_CIDR} \\
      --use-service-account-credentials=true \\
      --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    ### 4. **Configure the Kubernetes Scheduler**:
    
    - Move the `kube-scheduler` kubeconfig into place.
    
    ```bash
    sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
    ```
    
    - Create a systemd unit file for the `kube-scheduler` service.
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/kubernetes/kubernetes
    
    [Service]
    ExecStart=/usr/local/bin/kube-scheduler \\
      --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
      --leader-elect=true \\
      --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    **Secure kubeconfigs**
    
    ```
    sudo chmod 600 /var/lib/kubernetes/*.kubeconfig
    ```
    
    **Optional - Check Certificates and kubeconfigs**
    
    At `master-1` and `master-2` nodes, run the following, selecting option 3
    
    ```
    ./cert_verify.sh
    ```
    
    If output: 
    
    ```bash
    Checking /var/lib/kubernetes/kube-scheduler.kubeconfig
    /var/lib/kubernetes/kube-scheduler.kubeconfig found
    Path to CA certificate is correct
    Path to client certificate is correct
    Path to client key is correct
    Server URL  is incorrect
    
    vagrant@master-1:~$ cat /var/lib/kubernetes/kube-scheduler.kubeconfig
    apiVersion: v1
    clusters: null
    contexts:
    - context:
        cluster: kubernetes-the-hard-way
        user: system:kube-scheduler
      name: default
    current-context: default
    kind: Config
    preferences: {}
    users:
    - name: system:kube-scheduler
      user:
        client-certificate: /var/lib/kubernetes/pki/kube-scheduler.crt
        client-key: /var/lib/kubernetes/pki/kube-scheduler.key
    ```
    
    `Server URL  is incorrect` maybe error in `kube-scheduler.kubeconfig` file, 
    
    ```bash
    apiVersion: v1
    clusters: null
    contexts:
    - context:
        cluster: kubernetes-the-hard-way
        user: system:kube-scheduler
      name: default
    current-context: default
    kind: Config
    preferences: {}
    users:
    - name: system:kube-scheduler
      user:
        client-certificate: /var/lib/kubernetes/pki/kube-scheduler.crt
        client-key: /var/lib/kubernetes/pki/kube-scheduler.key
    ```
    
    fix:
    
    - Because api server is [`https://127.0.0.1:6443`](https://127.0.0.1:6443/)
    - The value for `<KUBERNETES_API_SERVER_ENDPOINT>` in the kube-scheduler configuration depends on where your kube-scheduler is running in relation to the Kubernetes API server. Here are the typical scenarios:
        1. **If the kube-scheduler is running on the same machine as the Kubernetes API server:** You can use `127.0.0.1:6443`. This local loopback address assumes that the kube-scheduler and the API server are on the same node, and it will connect to the API server on the local machine.
        2. **If the kube-scheduler is running on a different machine than the Kubernetes API server:** You should use the actual IP address or DNS name of the Kubernetes API server, like `https://192.168.56.11:6443`. This ensures that the kube-scheduler can reach the API server over the network.
        
        In most Kubernetes setups, especially in production or multi-node environments, the kube-scheduler is not typically on the same machine as the API server. In such cases, you need to use the actual network address. However, in single-node setups or for certain testing scenarios, it might be on the same node, and using `127.0.0.1:6443` would be appropriate.
        
    
    ```bash
    # INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    LOCAL_IP="127.0.0.1"
    cat <<EOF | sudo tee /var/lib/kubernetes/kube-scheduler.kubeconfig
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /var/lib/kubernetes/pki/ca.crt
        server: https://${LOCAL_IP}:6443
      name: kubernetes-the-hard-way
    contexts:
    - context:
        cluster: kubernetes-the-hard-way
        user: system:kube-scheduler
      name: default
    current-context: default
    kind: Config
    preferences: {}
    users:
    - name: system:kube-scheduler
      user:
        client-certificate: /var/lib/kubernetes/pki/kube-scheduler.crt
        client-key: /var/lib/kubernetes/pki/kube-scheduler.key
    EOF
    ```
    
    ### 5. **Start the Controller Services**:
    
    - Reload `systemd`, enable, and start the `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler` services.
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
    sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
    ```
    
    ### 6. **Verification**:
    
    - Verify the status of the control plane components using `kubectl get componentstatuses`.
    
    ```bash
    kubectl get componentstatuses --kubeconfig admin.kubeconfig
    ```
    
    ### Provisioning a Network Load Balancer (NLB)
    
    ### 1. **Install HAProxy**:
    
    - On the `loadbalancer` instance, install HAProxy, a software-based load balancer.
    
    ```bash
    sudo apt-get update && sudo apt-get install -y haproxy
    ```
    
    ### 2. **Configure HAProxy**:
    
    - Set up HAProxy to distribute incoming API server traffic evenly between the two master nodes.
    
    ```bash
    MASTER_1=$(dig +short master-1)
    MASTER_2=$(dig +short master-2)
    MASTER_3=$(dig +short master-3)
    LOADBALANCER=$(dig +short loadbalancer)
    ```
    
    ```bash
    cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
    frontend kubernetes
        bind ${LOADBALANCER}:6443
        option tcplog
        mode tcp
        default_backend kubernetes-master-nodes
    
    backend kubernetes-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        server master-1 ${MASTER_1}:6443 check fall 3 rise 2
        server master-2 ${MASTER_2}:6443 check fall 3 rise 2
        server master-3 ${MASTER_3}:6443 check fall 3 rise 2
    EOF
    ```
    
    ```bash
    sudo systemctl restart haproxy
    ```
    
    ### 3. **Verification**:
    
    - Make an HTTP request to the load balancer's IP to get the Kubernetes version, confirming the load balancer's functionality.
    
    ```bash
    curl [https://$](https://$/){LOADBALANCER}:6443/version -k
    ```
    
    ### Summary
    
    - You've bootstrapped the Kubernetes control plane for high availability across two nodes.
    - The external load balancer ensures that API server traffic is managed effectively.
    - Each control plane component is configured and verified to be operational.
    
    ### Next Steps
    
    - The next step in the lab series involves installing the Container Runtime Interface (CRI) on the Kubernetes worker nodes.
    
    ### Note
    
    - In a production environment, an odd number of master nodes is recommended for optimal etcd and control plane functionality.
    - Using `tmux` can be beneficial for running commands on multiple instances in parallel.
- 09: **Installing Container Runtime on the Kubernetes Worker Nodes**
    
    **Overview**
    
    - **Goal**: Install **`containerd`** as the container runtime on Kubernetes worker nodes.
    - **Why `containerd`**: It's a lightweight container runtime that is compatible with Kubernetes and replaces Docker.
    - **Additional Components**: CNI Plugins for container networking and **`runc`** for running containers.
    
    ### **Steps to Install Container Runtime**
    
    ### 1. **Set up Kubernetes `apt` Repository**:
    
    - Add the Kubernetes repository to your package manager to download Kubernetes packages.
    - Use **`curl`** to fetch the GPG key for the repository and add it to your system's trusted keys.
    
    ```bash
    {
      KUBE_LATEST=$(curl -L -s https://dl.k8s.io/release/stable.txt | awk 'BEGIN { FS="." } { printf "%s.%s", $1, $2 }')
    
      sudo mkdir -p /etc/apt/keyrings
      curl -fsSL https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    
      echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    }
    ```
    
    ---
    
    ### 2. **Install `containerd`, CNI Tools, and `kubectl`**:
    
    - Update your package lists (**`apt update`**).
    - Install **`containerd`**, Kubernetes CNI tools, **`kubectl`** (to help initialize kubeconfig files for worker-node auto-registration), **`ipvsadm`**, and **`ipset`**.
    
    ```bash
    {
      sudo apt update
      sudo apt install -y containerd kubernetes-cni kubectl ipvsadm ipset
    }
    ```
    
    ### 3. **Configure `containerd`**:
    
    - Create the **`containerd`** configuration directory.
    - Generate a default configuration file for **`containerd`** and modify it to enable systemd cgroups, which is required for Kubernetes.
    
    ```bash
    {
    sudo mkdir -p /etc/containerd
    containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
    }
    ```
    
    ### 4. **Restart `containerd`**:
    
    - Restart the **`containerd`** service to apply the new configuration.
    
    ```bash
    sudo systemctl restart containerd
    ```
    
    ### **Summary**
    
    - You've successfully installed and configured **`containerd`** as the container runtime on your Kubernetes worker nodes.
    - CNI plugins are installed to handle the networking for containers.
    - The **`runc`** binary, which is required to run containers, is included with **`containerd`**.
- 10: **Bootstrapping the Kubernetes Worker Nodes**
    
    In this lab, you'll be bootstrapping two Kubernetes worker nodes by setting up essential components like the `kubelet` and `kube-proxy`. These components are crucial for the worker nodes to function properly within your Kubernetes cluster.
    
    ### Overview
    
    - **Goal**: To set up and configure `kubelet` and `kube-proxy` on Kubernetes worker nodes.
    - **kubelet**: An agent that runs on each worker node, responsible for managing the pods and their containers.
    - **kube-proxy**: Manages network communication within the cluster, implementing part of the Kubernetes Service concept.
    
    ### Steps to Bootstrap the Kubernetes Worker Nodes
    
    ### 1. **Provision Kubelet Client Certificates**:
    
    - Create a certificate and key for each worker node (`worker-1`) on the `master-1` node.
    - These certificates must comply with the Kubernetes Node Authorizer requirements.
    
    On `master-1`:
    
    ```
    WORKER_1=$(dig +short worker-1)
    ```
    
    ```bash
    cat > openssl-worker-1.cnf <<EOF[req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name
    [req_distinguished_name]
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = worker-1
    IP.1 = ${WORKER_1}EOF
    
    openssl genrsa -out worker-1.key 2048
    openssl req -new -key worker-1.key -subj "/CN=system:node:worker-1/O=system:nodes" -out worker-1.csr -config openssl-worker-1.cnf
    openssl x509 -req -in worker-1.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out worker-1.crt -extensions v3_req -extfile openssl-worker-1.cnf -days 1000
    ```
    
    Results:
    
    ```
    worker-1.key
    worker-1.crt
    
    ```
    
    ### 2. **Generate the Kubelet Kubernetes Configuration File**:
    
    - Create a `kubeconfig` file for the kubelet on `worker-1`. This file contains credentials for the kubelet to communicate with the Kubernetes API server.
    
    ```bash
    LOADBALANCER=$(dig +short loadbalancer)
    ```
    
    ```bash
    {
      kubectl config set-cluster kubernetes-the-hard-way \
        --certificate-authority=/var/lib/kubernetes/pki/ca.crt \
        --server=https://${LOADBALANCER}:6443 \
        --kubeconfig=worker-1.kubeconfig
    
      kubectl config set-credentials system:node:worker-1 \
        --client-certificate=/var/lib/kubernetes/pki/worker-1.crt \
        --client-key=/var/lib/kubernetes/pki/worker-1.key \
        --kubeconfig=worker-1.kubeconfig
    
      kubectl config set-context default \
        --cluster=kubernetes-the-hard-way \
        --user=system:node:worker-1 \
        --kubeconfig=worker-1.kubeconfig
    
      kubectl config use-context default --kubeconfig=worker-1.kubeconfig
    }
    ```
    
    ### 3. **Transfer Certificates and Kubeconfig to the Worker Node**:
    
    - Copy the necessary certificates and `kubeconfig` files to `worker-1` using `scp`.
    
    ```bash
    scp ca.crt worker-1.crt worker-1.key worker-1.kubeconfig worker-1:~/
    ```
    
    ### 4. **Download and Install Worker Binaries**:
    
    - On `worker-1`, download the `kubelet` and `kube-proxy` binaries from the official Kubernetes release repository.
    - Install these binaries and create necessary directories for their operation.
    
    ```bash
    KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
    
    wget -q --show-progress --https-only --timestamping \
      https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kube-proxy \
      https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubelet
    ```
    
    ```bash
    sudo mkdir -p \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes/pki \
      /var/run/kubernetes
    ```
    
    ```bash
    {
    chmod +x kube-proxy kubelet
    sudo mv kube-proxy kubelet /usr/local/bin/
    }
    ```
    
    ### 5. **Configure the Kubelet**:
    
    - On `worker-1`, set up the kubelet configuration and systemd unit file.
    
    ```bash
    {
      sudo mv ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubernetes/pki/
      sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubelet.kubeconfig
      sudo mv ca.crt /var/lib/kubernetes/pki/
      sudo mv kube-proxy.crt kube-proxy.key /var/lib/kubernetes/pki/
      sudo chown root:root /var/lib/kubernetes/pki/*
      sudo chmod 600 /var/lib/kubernetes/pki/*
      sudo chown root:root /var/lib/kubelet/*
      sudo chmod 600 /var/lib/kubelet/*
    }
    ```
    
    ```bash
    POD_CIDR=10.244.0.0/16
    SERVICE_CIDR=10.96.0.0/16
    CLUSTER_DNS=$(echo $SERVICE_CIDR | awk 'BEGIN {FS="."} ; { printf("%s.%s.%s.10", $1, $2, $3) }')
    ```
    
    - Configure the kubelet to use the previously set up certificates and kubeconfig.
    
    ```bash
    cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    authentication:
      anonymous:
        enabled: false
      webhook:
        enabled: true
      x509:
        clientCAFile: /var/lib/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
    containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
    clusterDomain: cluster.local
    clusterDNS:
      - ${CLUSTER_DNS}
    cgroupDriver: systemd
    resolvConf: /run/systemd/resolve/resolv.conf
    runtimeRequestTimeout: "15m"
    tlsCertFile: /var/lib/kubernetes/pki/${HOSTNAME}.crt
    tlsPrivateKeyFile: /var/lib/kubernetes/pki/${HOSTNAME}.key
    registerNode: true
    EOF
    ```
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
    [Unit]
    Description=Kubernetes Kubelet
    Documentation=https://github.com/kubernetes/kubernetes
    After=containerd.service
    Requires=containerd.service
    
    [Service]
    ExecStart=/usr/local/bin/kubelet \\
      --config=/var/lib/kubelet/kubelet-config.yaml \\
      --kubeconfig=/var/lib/kubelet/kubelet.kubeconfig \\
      --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    ### 6. **Configure the Kubernetes Proxy**:
    
    - Set up the kube-proxy configuration and systemd unit file on `worker-1`.
    
    ```bash
    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/
    ```
    
    - This includes specifying the mode of operation and the path to the `kube-proxy` kubeconfig file.
    
    ```bash
    cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
    kind: KubeProxyConfiguration
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    clientConnection:
      kubeconfig: /var/lib/kube-proxy/kube-proxy.kubeconfig
    mode: ipvs
    clusterCIDR: ${POD_CIDR}
    EOF
    ```
    
    ```bash
    cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
    [Unit]
    Description=Kubernetes Kube Proxy
    Documentation=https://github.com/kubernetes/kubernetes
    
    [Service]
    ExecStart=/usr/local/bin/kube-proxy \\
      --config=/var/lib/kube-proxy/kube-proxy-config.yaml
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    **Optional - Check Certificates and kubeconfigs**
    
    At `worker-1` node, run the following, selecting option 4
    
    ```bash
    ./cert_verify.sh
    
    ```
    
    ### 7. **Start the Worker Services**:
    
    - Enable and start the `kubelet` and `kube-proxy` services on `worker-1`.
    
    ```bash
    {
      sudo systemctl daemon-reload
      sudo systemctl enable kubelet kube-proxy
      sudo systemctl start kubelet kube-proxy
    }
    ```
    
    ### Verification
    
    - **From the `master-1` Node**:
        - Verify that the worker node is registered with the cluster by running `kubectl get nodes`.
        
        ```bash
        kubectl get nodes --kubeconfig admin.kubeconfig
        ```
        
        - Initially, the worker node might show as `NotReady` since pod networking is not yet installed.
        
        ![Untitled](./Kubernetes/manual-setup/Untitled%201.png)
        
    
    ### Summary
    
    - The worker nodes are now bootstrapped with the kubelet and kube-proxy, which are essential for their operation within the Kubernetes cluster.
    - The node authorization and authentication are set up to comply with Kubernetes security standards.
    
    ### Next Steps
    
    - The next lab in the series will likely involve setting up networking for the pods on the worker nodes, which will change the node status from `NotReady` to `Ready`.
    
    ### Note
    
    - It's important to correctly set up the certificates and kubeconfigs, as they are crucial for the secure operation of the Kubernetes cluster.
    - Make sure to follow the same process for any additional worker nodes in the cluster.
- 11: **TLS Bootstrapping Worker Nodes**
    
    In this lab, you are implementing TLS bootstrapping for Kubernetes worker nodes. TLS bootstrapping is a process that allows kubelets to automatically request and renew certificates needed for secure communication with the Kubernetes API server. This process is critical for automating and scaling the management of certificates in large or dynamic Kubernetes clusters.
    
    ### Overview
    
    - **Goal**: Automate the generation and renewal of certificates for Kubernetes worker nodes.
    - **Why TLS Bootstrapping**: Manually generating and renewing certificates for each node is impractical in large clusters. TLS bootstrapping automates this process.
    
    ### Steps for TLS Bootstrapping Worker Nodes
    
    ### 1. **Certificates API**:
    
    - The Certificates API in Kubernetes allows kubelets to create and submit Certificate Signing Requests (CSRs) and retrieve signed certificates.
    
    ### 2. **Prerequisites**:
    
    - Ensure that bootstrap token-based authentication is enabled in the kube-apiserver.
    - The kube-controller-manager should have access to the CA certificates and keys to sign the CSRs.
    
    ### 3. **Create the Bootstrap Token**:
    
    - Generate a bootstrap token on the `master-1` node. This token will be used by kubelets to authenticate with the Kubernetes API server to submit CSRs.
    
    ```bash
    EXPIRATION=$(date -u --date "+7 days" +"%Y-%m-%dT%H:%M:%SZ")
    cat > bootstrap-token-07401b.yaml <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      # Name MUST be of form "bootstrap-token-<token id>"
      name: bootstrap-token-07401b
      namespace: kube-system
    
    # Type MUST be 'bootstrap.kubernetes.io/token'
    type: bootstrap.kubernetes.io/token
    stringData:
      # Human readable description. Optional.
      description: "The default bootstrap token generated by 'kubeadm init'."
    
      # Token ID and secret. Required.
      token-id: 07401b
      token-secret: f395accd246ae52d
    
      # Expiration. Optional.
      expiration: ${EXPIRATION}
    
      # Allowed usages.
      usage-bootstrap-authentication: "true"
      usage-bootstrap-signing: "true"
    
      # Extra groups to authenticate the token as. Must start with "system:bootstrappers:"
      auth-extra-groups: system:bootstrappers:worker
    EOF
    
    kubectl create -f bootstrap-token-07401b.yaml --kubeconfig admin.kubeconfig
    ```
    
    ### 4. **Authorize Workers to Create CSR**:
    
    - Create a ClusterRoleBinding to authorize worker nodes to create CSR requests.
        1. **SSH into the Master Node**: Log into **`master-1`** or any other master node in your Kubernetes cluster where you have administrative access.
        2. **Run the Commands**:
            - You can directly create a **`ClusterRoleBinding`** using the **`kubectl`** command:
                
                ```bash
                kubectl create clusterrolebinding create-csrs-for-bootstrapping \
                  --clusterrole=system:node-bootstrapper \
                  --group=system:bootstrappers \
                  --kubeconfig admin.kubeconfig
                
                ```
                
            - Or, alternatively, use a YAML file to define the **`ClusterRoleBinding`** and then apply it with **`kubectl`**:
                
                ```bash
                cat > csrs-for-bootstrapping.yaml <<EOF
                # enable bootstrapping nodes to create CSR
                kind: ClusterRoleBinding
                apiVersion: rbac.authorization.k8s.io/v1
                metadata:
                  name: create-csrs-for-bootstrapping
                subjects:
                - kind: Group
                  name: system:bootstrappers
                  apiGroup: rbac.authorization.k8s.io
                roleRef:
                  kind: ClusterRole
                  name: system:node-bootstrapper
                  apiGroup: rbac.authorization.k8s.io
                EOF
                
                kubectl create -f csrs-for-bootstrapping.yaml --kubeconfig admin.kubeconfig
                
                ```
                
            
            In both cases, ensure that you are using the correct kubeconfig file that grants you administrative access to the cluster.
            
        3. **Verification**: After running the command, you can verify the creation of the **`ClusterRoleBinding`** using:
            
            ```bash
            bashCopy code
            kubectl get clusterrolebinding create-csrs-for-bootstrapping --kubeconfig admin.kubeconfig
            
            ```
            
        
        By performing these steps, you are setting up the necessary permissions for worker nodes to bootstrap themselves into the Kubernetes cluster by creating their own CSRs, which is a key part of the TLS bootstrapping process.
        
    
    ### 5. **Authorize Workers to Approve CSRs**:
    
    - Create a ClusterRoleBinding to allow the `system:bootstrappers` group to approve CSR requests.
    
    ```
    kubectl create clusterrolebinding auto-approve-csrs-for-group \
      --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
      --group=system:bootstrappers \
      --kubeconfig admin.kubeconfig
    ```
    
    - -------------- OR ---------------
    
    ```bash
    cat > auto-approve-csrs-for-group.yaml <<EOF# Approve all CSRs for the group "system:bootstrappers"
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: auto-approve-csrs-for-group
    subjects:
    - kind: Group
      name: system:bootstrappers
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
      apiGroup: rbac.authorization.k8s.io
    EOF
    
    kubectl create -f auto-approve-csrs-for-group.yaml --kubeconfig admin.kubeconfig
    ```
    
    - **Verification**: After running the command, you can check the existence of the **`ClusterRoleBinding`** using:
        
        ```bash
        kubectl get clusterrolebinding auto-approve-csrs-for-group --kubeconfig admin.kubeconfig
        ```
        
    
    By completing this step, you set up the Kubernetes cluster to automatically approve CSR requests from worker nodes, facilitating their self-registration and certificate management. This is a key part of automating the management of TLS certificates in a Kubernetes cluster.
    
    ### 6. **Authorize workers(kubelets) to Auto Renew Certificates on expiration**
    
    - Set up another ClusterRoleBinding to allow nodes to automatically renew certificates upon expiration.
    - Log into **`master-1`** or another master node in your Kubernetes cluster.
    - **Run the Commands**:
        - You can create the ClusterRoleBinding directly using the **`kubectl`** command:
            
            ```bash
            kubectl create clusterrolebinding auto-approve-renewals-for-nodes \
              --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient \
              --group=system:nodes \
              --kubeconfig admin.kubeconfig
            ```
            
        - Alternatively, define the ClusterRoleBinding in a YAML file and apply it with **`kubectl`**:
            
            ```bash
            cat > auto-approve-renewals-for-nodes.yaml <<EOF
            # Approve renewal CSRs for the group "system:nodes"
            kind: ClusterRoleBinding
            apiVersion: rbac.authorization.k8s.io/v1
            metadata:
              name: auto-approve-renewals-for-nodes
            subjects:
            - kind: Group
              name: system:nodes
              apiGroup: rbac.authorization.k8s.io
            roleRef:
              kind: ClusterRole
              name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
              apiGroup: rbac.authorization.k8s.io
            EOF
            
            kubectl create -f auto-approve-renewals-for-nodes.yaml --kubeconfig admin.kubeconfig
            ```
            
    - **Verification**:
        - After running the command, verify the creation of the ClusterRoleBinding:
            
            ```bash
            kubectl get clusterrolebinding auto-approve-renewals-for-nodes --kubeconfig admin.kubeconfig
            ```
            
    
    By completing this step, you ensure that nodes in your Kubernetes cluster can automatically renew their certificates before they expire, which is key for maintaining ongoing, secure communication within the cluster. This step helps in reducing the manual intervention required for certificate management in a large or dynamic cluster.
    
    ### 7. **Configure Worker Node**:
    
    1. **Log into `worker-2` Node**: You need to SSH into your **`worker-2`** node where these commands will be executed.
        1. **Download Kubernetes Binaries**:
            - Determine the latest stable version of Kubernetes:
                
                ```bash
                KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
                ```
                
            - Download the **`kube-proxy`** and **`kubelet`** binaries for the determined version:
                
                ```bash
                wget -q --show-progress --https-only --timestamping \
                  https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kube-proxy \
                  https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubelet
                ```
                
        2. **Create Installation Directories**:
            - You need to create directories where Kubernetes will store its configurations, certificates, and other necessary files:
                
                ```bash
                sudo mkdir -p \
                  /var/lib/kubelet/pki \
                  /var/lib/kube-proxy \
                  /var/lib/kubernetes/pki \
                  /var/run/kubernetes
                ```
                
        3. **Install Worker Binaries**:
            - Change the downloaded files to executable and move them to a directory in the system's PATH:
                
                ```bash
                chmod +x kube-proxy kubelet
                sudo mv kube-proxy kubelet /usr/local/bin/
                ```
                
        4. **Move and Secure Certificates**:
            - Transfer the CA, kube-proxy certificates, and keys to the appropriate directories and set the correct permissions:
                
                ```bash
                sudo mv ca.crt kube-proxy.crt kube-proxy.key /var/lib/kubernetes/pki
                sudo chown root:root /var/lib/kubernetes/pki/*
                sudo chmod 600 /var/lib/kubernetes/pki/*
                ```
                
        
        By completing these steps, you have successfully installed the necessary binaries on your **`worker-2`** node and have securely placed the required certificates and keys. These actions are essential for the kubelet and kube-proxy to function correctly on the worker node.
        
    
    ### 8. **Start Worker Services**
    
    In Step 6 of the TLS bootstrapping process, you are configuring the `worker-2` node to use a bootstrap token for the initial communication with the Kubernetes API server. This process involves creating a `bootstrap-kubeconfig` file on `worker-2`. Here's a guide on how to do it:
    
    ### SSH into `worker-2` Node
    
    First, ensure you are logged into the `worker-2` node where you will perform these operations.
    
    ### Set up Shell Variables
    
    Set some necessary shell variables used in the configurations:
    
    ```bash
    LOADBALANCER=$(dig +short loadbalancer)
    POD_CIDR=10.244.0.0/16
    SERVICE_CIDR=10.96.0.0/16
    CLUSTER_DNS=$(echo $SERVICE_CIDR | awk 'BEGIN {FS="."} ; { printf("%s.%s.%s.10", $1, $2, $3) }')
    
    ```
    
    ### Create the Bootstrap Kubeconfig
    
    There are two methods to create the `bootstrap-kubeconfig`. You can choose either:
    
    1. **Using `kubectl` Commands**:
        - Run the following commands to set up the `bootstrap-kubeconfig`:
            
            ```bash
            sudo kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \\
              set-cluster bootstrap --server="<https://$>{LOADBALANCER}:6443" --certificate-authority=/var/lib/kubernetes/pki/ca.crt
            
            sudo kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \\
              set-credentials kubelet-bootstrap --token=07401b.f395accd246ae52d
            
            sudo kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \\
              set-context bootstrap --user=kubelet-bootstrap --cluster=bootstrap
            
            sudo kubectl config --kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig \\
              use-context bootstrap
            
            ```
            
    2. **Using a Here Document to Create the File**:
        - Use a here document to create the `bootstrap-kubeconfig` file directly:
            
            ```bash
            cat <<EOF | sudo tee /var/lib/kubelet/bootstrap-kubeconfig
            apiVersion: v1
            clusters:
            - cluster:
                certificate-authority: /var/lib/kubernetes/pki/ca.crt
                server: <https://$>{LOADBALANCER}:6443
              name: bootstrap
            contexts:
            - context:
                cluster: bootstrap
                user: kubelet-bootstrap
              name: bootstrap
            current-context: bootstrap
            kind: Config
            preferences: {}
            users:
            - name: kubelet-bootstrap
              user:
                token: 07401b.f395accd246ae52d
            EOF
            
            ```
            
    
    ### Reference
    
    For more details, you can refer to the Kubernetes documentation on [Kubelet TLS Bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/).
    
    By completing these steps, you have successfully configured the `worker-2` node to use TLS bootstrapping for its initial setup and communication with the Kubernetes cluster. This configuration is critical for enabling the node to automatically request and receive the necessary certificates to join and operate within the cluster securely.
    
    ## Step 7 Create Kubelet Config File
    
    Create the `kubelet-config.yaml` configuration file:
    
    Reference: [https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
    
    ```
    cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    authentication:
      anonymous:
        enabled: false
      webhook:
        enabled: true
      x509:
        clientCAFile: /var/lib/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
    containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
    cgroupDriver: systemd
    clusterDomain: "cluster.local"
    clusterDNS:
      - ${CLUSTER_DNS}registerNode: true
    resolvConf: /run/systemd/resolve/resolv.conf
    rotateCertificates: true
    runtimeRequestTimeout: "15m"
    serverTLSBootstrap: true
    EOF
    ```
    
    > Note: We are not specifying the certificate details - tlsCertFile and tlsPrivateKeyFile - in this file
    > 
    
    ## Step 8 Configure Kubelet Service
    
    Create the `kubelet.service` systemd unit file:
    
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
    [Unit]
    Description=Kubernetes Kubelet
    Documentation=https://github.com/kubernetes/kubernetes
    After=containerd.service
    Requires=containerd.service
    
    [Service]
    ExecStart=/usr/local/bin/kubelet \\  --bootstrap-kubeconfig="/var/lib/kubelet/bootstrap-kubeconfig" \\  --config=/var/lib/kubelet/kubelet-config.yaml \\  --kubeconfig=/var/lib/kubelet/kubeconfig \\  --cert-dir=/var/lib/kubelet/pki/ \\  --v=2
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    Things to note here:
    
    - **bootstrap-kubeconfig**: Location of the bootstrap-kubeconfig file.
    - **cert-dir**: The directory where the generated certificates are stored.
    - **kubeconfig**: We specify a location for this *but we have not yet created it*. Kubelet will create one itself upon successful bootstrap.
    
    ## Step 9 Configure the Kubernetes Proxy
    
    In one of the previous steps we created the kube-proxy.kubeconfig file. Check [here](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md) if you missed it.
    
    ```
    {
      sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/
      sudo chown root:root /var/lib/kube-proxy/kube-proxy.kubeconfig
      sudo chmod 600 /var/lib/kube-proxy/kube-proxy.kubeconfig
    }
    ```
    
    Create the `kube-proxy-config.yaml` configuration file:
    
    Reference: [https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/](https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/)
    
    ```
    cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
    kind: KubeProxyConfiguration
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    clientConnection:
      kubeconfig: /var/lib/kube-proxy/kube-proxy.kubeconfig
    mode: ipvs
    clusterCIDR: ${POD_CIDR}EOF
    ```
    
    Create the `kube-proxy.service` systemd unit file:
    
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
    [Unit]
    Description=Kubernetes Kube Proxy
    Documentation=https://github.com/kubernetes/kubernetes
    
    [Service]
    ExecStart=/usr/local/bin/kube-proxy \\  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    EOF
    ```
    
    ## Step 10 Start the Worker Services
    
    On worker-2:
    
    ```
    {
      sudo systemctl daemon-reload
      sudo systemctl enable kubelet kube-proxy
      sudo systemctl start kubelet kube-proxy
    }
    ```
    
    > Remember to run the above commands on worker node: worker-2
    > 
    
    ### Optional - Check Certificates and kubeconfigs
    
    At `worker-2` node, run the following, selecting option 5
    
    ```
    ./cert_verify.sh
    
    ```
    
    ## Step 11 Approve Server CSR
    
    Now, go back to `master-1` and approve the pending kubelet-serving certificate
    
    [//](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/docs/11-tls-bootstrapping-kubernetes-workers.md#): # (command:kubectl certificate approve --kubeconfig admin.kubeconfig $(kubectl get csr --kubeconfig admin.kubeconfig -o json | jq -r '.items | .[] | select(.spec.username == "system:node:worker-2") | .metadata.name'))
    
    ```
    kubectl get csr --kubeconfig admin.kubeconfig
    ```
    
    > Output - Note the name will be different, but it will begin with csr-
    > 
    
    ```
    NAME        AGE   SIGNERNAME                                    REQUESTOR                 REQUESTEDDURATION   CONDITION
    csr-7k8nh   85s   kubernetes.io/kubelet-serving                 system:node:worker-2      <none>              Pending
    csr-n7z8p   98s   kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:07401b   <none>              Approved,Issued
    
    ```
    
    Approve the pending certificate. Note that the certificate name `csr-7k8nh` will be different for you, and each time you run through.
    
    ```
    kubectl certificate approve --kubeconfig admin.kubeconfig csr-7k8nh
    
    ```
    
    Note: In the event your cluster persists for longer than 365 days, you will need to manually approve the replacement CSR.
    
    Reference: [https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#kubectl-approval](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#kubectl-approval)
    
    ## Verification
    
    List the registered Kubernetes nodes from the master node:
    
    ```
    kubectl get nodes --kubeconfig admin.kubeconfig
    ```
    
    > output
    > 
    
    ```
    NAME       STATUS      ROLES    AGE   VERSION
    worker-1   NotReady    <none>   93s   v1.28.4
    worker-2   NotReady    <none>   93s   v1.28.4
    ```
    
- 12: **Configuring kubectl for Remote Access**
    
    In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.
    
    > Run the commands in this lab from the same directory used to generate the admin client certificates.
    > 
    
    ## The Admin Kubernetes Configuration File
    
    Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.
    
    On `master-1`
    
    Get the kube-api server load-balancer IP.
    
    ```bash
    LOADBALANCER=$(dig +short loadbalancer)
    
    ```
    
    Generate a kubeconfig file suitable for authenticating as the `admin` user:
    
    ```bash
    {
    
      kubectl config set-cluster kubernetes-the-hard-way \\
        --certificate-authority=ca.crt \\
        --embed-certs=true \\
        --server=https://${LOADBALANCER}:6443
    
      kubectl config set-credentials admin \\
        --client-certificate=admin.crt \\
        --client-key=admin.key
    
      kubectl config set-context kubernetes-the-hard-way \\
        --cluster=kubernetes-the-hard-way \\
        --user=admin
    
      kubectl config use-context kubernetes-the-hard-way
    }
    
    ```
    
    Reference doc for kubectl config [here](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
    
    ## Verification
    
    Check the health of the remote Kubernetes cluster:
    
    ```
    kubectl get componentstatuses
    
    ```
    
    > output
    > 
    
    ```
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS    MESSAGE             ERROR
    controller-manager   Healthy   ok
    scheduler            Healthy   ok
    etcd-1               Healthy   {"health":"true"}
    etcd-0               Healthy   {"health":"true"}
    
    ```
    
    List the nodes in the remote Kubernetes cluster:
    
    ```bash
    kubectl get nodes
    
    ```
    
    > output
    > 
    
    ```
    NAME       STATUS      ROLES    AGE    VERSION
    worker-1   NotReady    <none>   118s   v1.28.4
    worker-2   NotReady    <none>   118s   v1.28.4
    
    ```
    
    If the result is:
    
    ```bash
    vagrant@master-1:~$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS      MESSAGE                                                                                        ERROR
    scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused   
    controller-manager   Healthy     ok                                                                                             
    etcd-0               Healthy     ok
    ```
    
    Fix:
    
    ```bash
    vagrant@master-1:~$ # INTERNAL_IP=$(ip addr show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f 1)
    LOCAL_IP="127.0.0.1"
    cat <<EOF | sudo tee /var/lib/kubernetes/kube-scheduler.kubeconfig
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /var/lib/kubernetes/pki/ca.crt
        server: https://${LOCAL_IP}:6443
      name: kubernetes-the-hard-way
    contexts:
    - context:
        cluster: kubernetes-the-hard-way
        user: system:kube-scheduler
      name: default
    current-context: default
    kind: Config
    preferences: {}
    users:
    - name: system:kube-scheduler
      user:
        client-certificate: /var/lib/kubernetes/pki/kube-scheduler.crt
        client-key: /var/lib/kubernetes/pki/kube-scheduler.key
    EOF
    
    vagrant@master-1:~$ kubectl get componentstatuses --kubeconfig admin.kubeconfig
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS    MESSAGE   ERROR
    controller-manager   Healthy   ok        
    etcd-0               Healthy   ok        
    scheduler            Healthy   ok
    ```
    
- 13: **Provisioning Pod Network**
    
    [](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/docs/13-configure-pod-networking.md)
    
- 14: **RBAC for Kubelet Authorization**