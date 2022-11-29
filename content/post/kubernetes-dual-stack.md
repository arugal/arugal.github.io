---
title: "搭建一个 IPv4/IPv6 双栈网络 Kubernetes 集群"
date: 2022-11-25T12:35:19+08:00
draft: false
ShowToc: false
---

### 概述

本文讲述如何使用 kubeadm 在腾讯云上搭建一个 IPv4/IPv6 双栈网络 Kuberentes 集群，全文使用到的项目如下:

+ Docker 20.10.21

+ Kuberentes v1.23.14

+ Calico v3.24.5

  ![dula-stack](https://cdn.jsdelivr.net/gh/arugal/img@main/image/kuberentes-dual-stack.png)

### 前期准备

1. 在腾讯云购买两台云服务器, 服务器需要启用 IPv6

   |      机器       |     IPv4     |                 IPv6                 |    说明     |
   | :-------------: | :----------: | :----------------------------------: | :---------: |
   | VM-16-9-centos  | 10.206.16.9  | 2402:4e00:1801:d000:0:97da:5b30:d4c3 | Master 节点 |
   | VM-16-15-centos | 10.206.16.15 | 2402:4e00:1801:d000:0:97da:5b09:8905 | Worker 节点 |

2. 系统配置

   ```bash
   # 关闭防火墙
   systemctl stop firewalld
   systemctl disable firewalld
   
   # 禁用 SELinux
   sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
   setenforce 0
   getenforce
   
   # 禁用交换分区 (云服务器默认已经禁用)
   swapoff -a
   sed -i /^[^#]*swap*/s/^/\#/g /etc/fstab
   
   # 设置内核参数
   echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
   echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
   
   sysctl -p

3. 安装 docker

   ```bash
   # 安装 docker
   yum install -y yum-utils && \
   yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && \
   yum install -y docker-ce docker-ce-cli containerd.io 
   
   # 修改 daemon.json
   mkdir -p /etc/docker
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"]
   }
   EOF
   
   # 重启 docker
   systemctl enable docker
   systemctl restart docker
   ```

4. 安装 kubeadm、kubelet 和 kubectl

   ```bash
   # 添加 kuberentes 阿里云源
   cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   
   # 安装 kubeadm、kubelet 和 kubectl
   yum install -y --nogpgcheck kubelet-1.23.14-0 kubeadm-1.23.14-0 kubectl-1.23.14-0
   
   # 启动 kubelet
   systemctl enable --now kubelet
   
   ```


### 安装

1. 初始化 Master 节点

   ```yaml
   ---
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: ClusterConfiguration
   networking:
     podSubnet: 10.244.0.0/16,2001:db8:42:0::/56
     serviceSubnet: 10.96.0.0/16,2001:db8:42:1::/112
   imageRepository: "registry.cn-hangzhou.aliyuncs.com/google_containers"
   kubernetesVersion: "v1.23.14"
   ---
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: InitConfiguration
   localAPIEndpoint:
     advertiseAddress: "10.206.16.9"
     bindPort: 6443
   nodeRegistration:
     kubeletExtraArgs:
       node-ip: 10.206.16.9,2402:4e00:1801:d000:0:97da:5b30:d4c3
     taints: []
   ```

   将以上内容复制到 master 节点的 `init.yaml ` 文件中, 然后执行 `kubadm init --config init.yaml` 初始化 Master 节点

2. 安装 Calico (使用 operator 的方式安装)

   a. 安装 tigera operator

      ```bash
      # 部署 tigera operator
      kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml
      ```

   b. 安装 calico

      ```
      # This section includes base Calico installation configuration.
      # For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
      apiVersion: operator.tigera.io/v1
      kind: Installation
      metadata:
        name: default
      spec:
        # Configures Calico networking.
        calicoNetwork:
          # Note: The ipPools section cannot be modified post-install.
          ipPools:
            - blockSize: 26
              cidr: 10.244.0.0/16
              encapsulation: IPIP
              natOutgoing: Enabled
              nodeSelector: all()
            - blockSize: 122
              cidr: 2001:db8:42:0::/56
              encapsulation: VXLANCrossSubnet
              natOutgoing: Enabled
              nodeSelector: all()
      ---
      
      # This section configures the Calico API server.
      # For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
      apiVersion: operator.tigera.io/v1
      kind: APIServer 
      metadata: 
        name: default 
      spec: {}
      ```

      将以上内容复制到 master 节点的 `custom-resources.yaml` 文件中，然后执行 `kubectl apply -f custom-resources.yaml`

3. 初始化 Worker 节点

   > 在 master 节点执行 `kubeadm token create --print-join-command` 命令获取 token 和 caCertHashes

   ```yaml
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: JoinConfiguration
   discovery:
     bootstrapToken:
       apiServerEndpoint: 10.206.16.9:6443
       token: "uufww4.on27548bh3yomo9e" # 替换成当前的 token
       caCertHashes:
       - "sha256:f75e342b7e77c2b276bd2490b352d3ea9253de33c9ce5610798a05b5739323e1" # 替换成当前的 caCertHashes
       # change auth info above to match the actual token and CA certificate hash for your cluster
   nodeRegistration:
     kubeletExtraArgs:
       node-ip: 10.206.16.1,2402:4e00:1801:d000:0:97da:5b09:8905
   ```


### 测试

1. 创建测试工作负载

   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
     labels:
       app: nginx
   spec:
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
           - name: nginx
             image: nginx:latest
             ports:
               - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
     labels:
       app: nginx
   spec:
     selector:
       app: nginx
     ipFamilyPolicy: PreferDualStack
     ports:
       - port: 80
         targetPort: 80
         protocol: TCP
     type: NodePort
   ```

2. 查看 Pod 和 Service 的 IP

   ![Pod 的 IPv4 和 IPv6](https://cdn.jsdelivr.net/gh/arugal/img@main/image/20221128220758.png)   

	 ![Service 的 IPv4 和 IPv6](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282208619.png)
	 

3. 通过 Pod 的 IPv4 和 IPv6 访问

   ![通过 Pod 的 IPv4 访问](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282209007.png)

   ![通过 Pod 的 IPv6 访问](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282212515.png)

4. 通过 Service 的 IPv4 和 IPv6 访问

   ![通过 Service 的 IPv4 访问](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282211883.png)

   ![通过 Service 的 IPv6 访问](https://cdn.jsdelivr.net/gh/arugal/img@main/image/image-20221128221303524.png)

5. 通过 NodePort 访问

   ![通过 IPv4 访问 NodePort](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282215439.png)

   ![通过 IPv6 访问 NodePort](https://cdn.jsdelivr.net/gh/arugal/img@main/image/202211282225862.png)
   
### 参考

1. [搭建 IPv6 私有网络](https://cloud.tencent.com/document/product/1142/47665)
2. [Linux 云服务器配置 IPv6](https://cloud.tencent.com/document/product/1142/47666)
3. [使用 kubeadm 支持双协议栈](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/dual-stack-support/)
4. [IPv4/IPv6 双协议栈](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dual-stack/)
5. [Configure dual stack or IPv6 only](https://projectcalico.docs.tigera.io/networking/ipv6#value)