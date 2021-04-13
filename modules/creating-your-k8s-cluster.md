# Module 5: Creating your Kubernetes Cluster using kubeadm

**Goal:** In this module, you will set up a Kubernetes cluster using `kubeadm`.

## Steps

Before going through this module, we need to make sure we have planned our kubernetes network IP ranges. We need to have an IP range for the pods and an IP range for services. For the sake of this lab we will be using the following:

```bash
POD CIDR == 172.16.0.0/16
SERVICE CIDR == 192.168.0.0/16
```

>You will find some configuration files and `demo` directory on the `master` node under `/home/calico-fortinet/` directory. All the configurations needed are in this directory and the demo app is under the `demo` directory. 

```text
|-- configs
|   |-- 0-install-kubeadm.sh
|   |-- 1-kubeadm-init-config.yaml
|   |-- 1-kubeadm-join-config.yaml
|   |-- 2-ebs-storageclass.yaml
|   |-- 3-loadbalancer.yaml
|   |-- 4-firewall-config.yaml
|   |-- dockerjsonconfig.json
|   `-- license.yaml
|-- demo
|   |-- storefront-demo.yaml
|   `-- tiers-demo.yaml
```

1. SSH into the `master` node, then update the `1-kubeadm-init-config.yaml` to add the FQDN of the master node(e.g `ip-10-99-2-246.us-west-2.compute.internal`). You can get it using `$ hostname -f`. Now you can create a new cluster configruation file based on this config.

    ```bash
    $ hostname -f

    ip-10-99-1-X.us-west-2.compute.internal

    $ cat 1-kubeadm-init-config.yaml

    apiVersion: kubeadm.k8s.io/v1beta2
    bootstrapTokens:
    - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
    kind: InitConfiguration
    localAPIEndpoint:
    bindPort: 6443
    nodeRegistration:
    criSocket: /var/run/dockershim.sock
    name: ip-10-99-1-X.us-west-2.compute.internal      # UPDATE THIS WITH THE MASTER FQDN
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
    kubeletExtraArgs:
      "feature-gates": "EphemeralContainers=true"
      cloud-provider: aws
    ---
    apiVersion: kubeadm.k8s.io/v1beta2
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    dns:
    type: CoreDNS
    etcd:
    local:
      dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.19.0
    networking:
    serviceSubnet: 192.168.0.0/16
    podSubnet: 172.16.0.0/16
    dnsDomain: cluster.local
    apiServer:
    timeoutForControlPlane: 4m0s
    extraArgs:
      "feature-gates": "EphemeralContainers=true"
      cloud-provider: aws
    scheduler:
    extraArgs:
      "feature-gates": "EphemeralContainers=true"
    controllerManager:
    extraArgs:
      "feature-gates": "EphemeralContainers=true"
      cloud-provider: aws
      configure-cloud-routes: "false"
    ```

2. Now you can launch kubernetes using kubeadm (On the `master` node)

    ```bash
    $ sudo kubeadm init --config 1-kubeadm-init-config.yaml

    W0923 20:55:22.992192    8924 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
    [init] Using Kubernetes version: v1.19.0
    [preflight] Running pre-flight checks
            [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
            [WARNING Hostname]: hostname "master" could not be reached
            [WARNING Hostname]: hostname "master": lookup master on 127.0.0.53:53: server misbehaving
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [192.168.0.1 10.99.1.203 54.200.135.157]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [10.99.1.203 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [10.99.1.203 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "kubelet.conf" kubeconfig file
    [kubeconfig] Writing "controller-manager.conf" kubeconfig file
    [kubeconfig] Writing "scheduler.conf" kubeconfig file
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Starting the kubelet
    [control-plane] Using manifest folder "/etc/kubernetes/manifests"
    [control-plane] Creating static Pod manifest for "kube-apiserver"
    [control-plane] Creating static Pod manifest for "kube-controller-manager"
    [control-plane] Creating static Pod manifest for "kube-scheduler"
    [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
    [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
    [apiclient] All control plane components are healthy after 11.002592 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Skipping phase. Please see --upload-certs
    [mark-control-plane] Marking the node master as control-plane by adding the label "node-role.kubernetes.io/master=''"
    [mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: abcdef.0123456789abcdef
    [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 10.99.1.203:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:c211c95124bde99ce8f78ae4b5fc0058d0d49c847b73e34764f1ae05f205b1d4
    ```

    Note the `sha256` hash value as you will need it when joining the worker nodes to the cluster.

3. Make sure you follow the below steps to setup `kubectl` and take note/save of the `kubeadm join ` coomand that was provided.

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

4. Verify that you can issue `kubectl` commands. The `master` node will be in the `NotReady` state until Calico us deployed.

    ```bash
    $ kubectl get nodes

    NAME                                        STATUS   ROLES    AGE   VERSION
    ip-10-99-1-150.us-west-2.compute.internal   NotReady    master   20h   v1.19.3
    ```

[Next -> Module 6](../modules/join-nodes.md)
