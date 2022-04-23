---
title: Install Kubernetes with Cilium on RHEL 8
tags: kubernetes k8s docker rhel linux cilium instructional
excerpt: "This is an instructional article that describes an install process of Kubernetes, using Cilium as it's networking CNI, onto Red Hat Enterprise Linux 8 hosts, using a one control plane, two worker node design." 
---

## Prerequisites

This article will assume that three Red Hat 8 systems will be employed, however the process is rather similar, if not the same at times, with the exceptions of OS-specific syntax (e.g package manners), and that these systems are freshly installed systems. 

However, I cannot promise this exact operation will work on systems which are vastly different, e.g Debian, your mileage may vary and you may be required to consult the relevant documentation for the operating system you are operating.

## Methodology

### Preparation & Installation

1. **Reconfigure SELinux to 'Permissive' on all systems**  

    On all systems, run

    ```bash
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

2. **Disable Swap**  

    Run the following command on all systems participating in the cluster, then remove the line containing the swap   partition configuration listed in `/etc/fstab`  
    The swapoff command will avoid the need for a reboot
  
    ```bash
    sudo swapoff -a
    ```

3. **Disable FirewallD** [^1]  

    ```bash
    systemctl disable --now firewalld
    ```

4. **Enable br\_netfilter module**  

    Configure the module into `modules-load.d` configuration file 

    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    br_netfilter
    EOF
    ```

    Configure iptable modules 

    ```bash
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF
    ```

    Apply sysctl configuration 

    ```bash
    sudo sysctl --system
    ```

5. **Install Docker on both worker & control plane nodes**  

    Follow the instructional I've written listed here: [Install Docker on RHEL 8](install-docker-on-rhel-8.html) on all systems, then on all systems, configure Docker to use systemd as the cgroup driver.

    Run the following command to create a `daemon.json` docker configuration file located at `/etc/docker/daemon.json` with the required parameters relating to cgroup drivers already pre-configured.

    ```bash
    cat <<EOF | sudo tee /etc/docker/daemon.json
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "local",
        "log-opts": {"max-size": "10m", "max-file": "3"}
    }
    EOF
    ```
  
   Once complete, restart docker using `sudo systemctl restart docker` on each system, worker and control-plane, then proceed to the next step. 
         
6. **Add the Kubernetes Repository to the package manager**  

    Run the following command to add the Kubernetes repository to the package manager.  

    ```bash
    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/docrpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF
    ```

7. **Install `kubelet`, `kubeadm` & `kubectl`, then start `kubelet` service**  

    ```bash
    sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    ```

    ```bash
    sudo systemctl enable --now kubelet
    ```

    _**Ensure this process thus far has been repeated for each system participating in the cluster, for both worker & control plane roles**_.  

8. **Install Helm**  

    On the control plane system, install Helm:  
        
    ```bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```


### Configuration

#### Kubernetes

On the control plane system, undertake the following:  

1. **Create `kubeadm-config.yml` file**  [^2]

    ```bash
    vim ~/kubeadm-config.yml
    ```

    ---

    ```yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: InitConfiguration
    ---
    apiVersion: kubeadm.k8s.io/v1beta3
    clusterName: cluster1
    kind: ClusterConfiguration
    kubernetesVersion: 1.23.0
    apiServer:
      certSANs:
        - 127.0.0.1
        - api.cluster1.k8s.domain.tld
    controllerManager:
      extraArgs:
        bind-address: 0.0.0.0
    scheduler:
      extraArgs:
        bind-address: 0.0.0.0
    featureGates:
      IPv6DualStack: true
    networking:
      dnsDomain: cluster1.k8s.domain.tld
      podSubnet: 10.64.0.0/14,fd00::/48
      serviceSubnet: 10.72.0.0/14,fd00:1::/108
    ---
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    ```

    ---

2.  **Initialise Kubernetes**  

    1. **Apply the `kubeadm-config.yml` file using `kubeadm`**  
    
        *Note: We're disabling the usage of kube-proxy, as we'll rely on Cillium for that functionality.*

        ```yaml
        kubeadm init --skip-phases=addon/kube-proxy --config kubeadm-config.yaml
        ```

        If successful, you should see an output that's similar to the following:

        ```bash
        Your Kubernetes control-plane has initialized successfully!

        To start using your cluster, you need to run the following as a regular user:

          mkdir -p $HOME/.kube
          sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
          sudo chown $(id -u):$(id -g) $HOME/.kube/config

        You should now deploy a Pod network to the cluster.
        Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        
          /docs/concepts/cluster-administration/addons/

        You can now join any number of machines by running the following on each node
        as root:

        kubeadm join <control-plane-host>:<control-plane-port> --token <token>--discovery-token-ca-cert-hash      sha256:<hash>
        ```


    2. **Follow the instruction listed in the output**  
              
        ```bash
        To start using your cluster, you need to run the following as a regular user
              
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
        ```
                  
    3.  **Check that Kubernetes is responding:**  

        ```bash
        kubectl get nodes 
        ```

        You should see the control plane listed.
    
    4. **Join the worker nodes to the cluster**

        On the worker nodes, proceed to join those nodes to the control plane node. Run the output akin to the following,   on each worker node.

        ```bash
        kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash   sha256:<hash>  
        ```
  
## Cilium Installation

Cilium is pretty straight-foward to install, with `helm` parsing either a YAML configuration file, or command line parameters, to integrate an installation of Cilium onto Kubernetes.  
  
On the control plane, run the following to install Cillium. Tweak values listed following the `--set` arguments to your liking. You can find further configuration variables listed oner at [https://docs.cilium.io/en/v1.9/helm-reference/#id1](https://docs.cilium.io/en/v1.9/helm-reference/#id1 "Helm Reference")
  
```bash
helm install cilium cilium/cilium --namespace kube-system --version 1.10.5 --namespace kube-system --set kubeProxyReplacement=strict --set k8sServiceHost=10.255.254.2 --set k8sServicePort=6443 --set ipam.mode=cluster-pool --set global.prometheus.enabled=true --set global.operatorPrometheus.enabled=true --set global.hubble.enabled=true --set global.hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" --set hubble.enabled=true --set hubble.listenAddress=":4244" --set hubble.relay.enabled=true --set hubble.ui.enabled=true --set nodeinit.enabled=true --set kubeProxyReplacement=partial --set externalIPs.enabled=true --set nodePort.enabled=true --set hostPort.enabled=true --set ipv6.enabled=true --set ipv4.enabled=true
```
  
If required, uninstall Cilium by running 
```bash
helm uninstall cilium -n kube-system
```

There is also `helm upgrade` which accepts the same syntax and arguments as `helm install`, which can be used to update Cilium and/or to reconfigure Cilium without removing Cilium beforehand. 

## Conclusion
  
You should now have a barebones Kubernetes install, which is running Cillium as it's networking CNI, ready for deploying containers into. The above does not configure any external networking or BGP peering for service-based networking, a future article will be written on using Cilium's BGP featureset (currently in beta) to advertise /32 addresses defined within a kubernetes service configuration. For now, more information can be found over at [Cilium - BGP (beta)](https://docs.cilium.io/en/v1.10/gettingstarted/bgp/) 

---

[^1]: Cilium will handle the networking. If required, one can further configure network policy within Cilium, however that is outside the scope of this article.

[^2]:   Note that this configurations configures both IPv6 and IPv4, which may cause connectivity issues if IPv6 end-to-end connectivity isn't yet already established between all nodes. Be sure to disable IPv6 in this configuration if this is the case, by redefining IPv6DualStack to `false`