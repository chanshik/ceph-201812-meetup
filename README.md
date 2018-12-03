# Ceph with Rook Hands-on

Ceph 분산 파일 시스템을 Rook Operator 를 이용해 VM 위에 직접 구축해보는 실습을 진행합니다.

- Vagrant 와 Virtualbox 를 이용해 Kubernetes 클러스터 구성
- Rook Operator 설치 및 클러스터 환경 설정
- Ceph 분산 파일 시스템 설치
- 어플리케이션을 Ceph 와 연결하여 배포



## Used packages in this hands-on

- Kubernetes (http://kubernetes.io)
- Docker (https://www.docker.com/)
- Rook (https://rook.io)
- Ceph (https://ceph.com)



# Setup Environments

- Host OS: Ubuntu 16.04
- Guest OS: Ubuntu 16.04 (ubuntu/xenial64) / 18.04 (ubuntu/bionic64)
- Automation Tool: Vagrant



## Install Virtualbox

사용하는 운영체제에 맞는 패키지를 받아 설치합니다.

- https://www.virtualbox.org/wiki/Downloads

```bash
sudo apt install virtualbox
```



## Install Vagrant 

VM 을 생성하면서 기본적인 초기화를 진행할 때 사용할 Vagrant 프로그램을 설치합니다.

- https://www.vagrantup.com/downloads.html

```bash
sudo dpkg -i vagrant_2.2.1_x86_64.deb
```



## Downloads Vagrant box image

Vagrant 를 이용해 VM 을 생성할 때 사용할 Box 파일을 미리 받아 디스크에 저장해둡니다.

Ubuntu 16.04 혹은 18.04 이미지를 이용합니다.

```bash
vagrant box add ubuntu/bionic64
```



## Downloads worksheet

github 저장소에 실습을 진행하면서 사용할 파일을 디렉토리별로 구분하여 저장해두었습니다.

```bash
git clone https://github.com/chanshik/ceph-201812-meetup.git
cd ceph-201812-meetup
ceph-201812-meetup$ 
```



## VM Networks

VM 에 할당한 IP 와 역할은 다음과 같습니다.

| Node  | IP         | Role   | Devices            |
| ----- | ---------- | ------ | ------------------ |
| k8s-1 | 10.254.1.2 | Master | /dev/sdc, /dev/sdd |
| k8s-2 | 10.254.1.3 | Worker | /dev/sdc, /dev/sdd |
| k8s-3 | 10.254.1.4 | Wokrer | /dev/sdc, /dev/sdd |



## Start VMs

미리 작성해둔 **Vagrantfile** 을 이용해 VM 3대를 시작합니다. 사용하는 장비 사양에 맞도록 CPU, 메모리, 추가 디스크 공간을 지정합니다.

실습에 필요한 환경을 구축하기 위해 VM 세 대를 생성합니다. 각 VM 은 시스템 디스크 외에 두 개의 디스크를 더 가지고 있습니다. 추가해둔 **/dev/sdc**, **/dev/sdd** 디스크를 **Ceph** 에서 **BlueStore** 로 활용합니다.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.box_check_update = false
  node_subnet = "10.254.1"

  (1..3).each do |i|
    config.vm.define "k8s-#{i}" do |node|
      node.vm.hostname = "k8s-#{i}"
      node.vm.network "private_network", ip: "#{node_subnet}.#{i + 1}"

      attached_disk_a = "disk-k8s-#{i}-a.vdi"
      attached_disk_b = "disk-k8s-#{i}-b.vdi"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "k8s-#{i}"
        vb.gui = false

        vb.cpus = 2
        vb.memory = "4096"

        unless File.exists?(attached_disk_a)
          vb.customize [
            'createhd', '--filename', attached_disk_a,
            '--variant', 'Fixed',
            '--size', 10 * 1024]
        end

        unless File.exists?(attached_disk_b)
          vb.customize [
            'createhd', '--filename', attached_disk_b,
            '--variant', 'Fixed',
            '--size', 10 * 1024]
        end

        vb.customize [
          'storageattach', :id, '--storagectl', 'SCSI',
          '--port', 2, '--device', 0, '--type', 'hdd',
          '--medium', attached_disk_a]

        vb.customize [
          'storageattach', :id, '--storagectl', 'SCSI',
          '--port', 3, '--device', 0, '--type', 'hdd',
          '--medium', attached_disk_b]
      end

      node.vm.provision "bootstrap", type: "shell", inline: <<-SHELL
        sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        sudo bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF'
        sudo apt update
        sudo apt install -y docker.io kubelet kubeadm kubectl ntp
        sudo usermod -aG docker vagrant

        sudo sed -i '/k8s/d' /etc/hosts
        sudo echo "#{node_subnet}.#{i + 1} k8s-#{i}" | sudo tee -a /etc/hosts
      SHELL
    end
  end
end
```

앞에서 작성한 **Vagrantfile** 을 이용해 VM 을 생성합니다.

```bash
vagrant up
```

VM 생성이 모두 끝난 다음에 ssh 를 실행하여 원하는 노드에 접속합니다.

```bash
vagrant ssh k8s-1
```



# Setup Kubernetes Cluster

## Select pod network add-on

Kubernetes 에서 사용할 CNI (Container Network Interface) 선택하고 **kubeadm** 을 이용해 초기화 할 때 같이 지정합니다. 실습에서는 **Calico** CNI 를 사용합니다.

- kubeadm 은 CNI 기반 네트워크만 지원
- Calico CIDR: 192.168.0.0/16
- https://kubernetes.io/docs/concepts/cluster-administration/addons/



## Initialize master node

Master node 에서 **kubeadm init** 명령을 실행하여 클러스터 초기화 작업을 시작합니다.

```bash
sudo swapoff -a
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.254.1.2

[init] using Kubernetes version: v1.12.2
[preflight] running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
...
[bootstraptoken] using token: s9qd0j.beetbemlhmmx1etd
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.254.1.2:6443 --token dzjclo.a8d0kjwcc64r7kvs --discovery-token-ca-cert-hash sha256:ce7c94f7863dbc1ad8d32028cb5388e4ea47a12959317d035b722e2a4fb3e5f3
```



## Add nodes

Master node 초기화 이후에는 추가하려는 노드에서 **kubeadm join** 명령을 실행합니다.

**@k8s-2**

```bash
sudo swapoff -a
sudo kubeadm join 10.254.1.2:6443 --token s9qd0j.beetbemlhmmx1etd --discovery-token-ca-cert-hash sha256:573bf08c800f2c9736d9b1b8a66421777dcd9e8991a2b9e0d7612c248bcdcdc5
```

**@k8s-3**

```bash
sudo swapoff -a
sudo kubeadm join 10.254.1.2:6443 --token s9qd0j.beetbemlhmmx1etd --discovery-token-ca-cert-hash sha256:573bf08c800f2c9736d9b1b8a66421777dcd9e8991a2b9e0d7612c248bcdcdc5
```



## Show kubernetes nodes

위 과정을 거쳐 생성한 Kubernetes 에 접근하려면 /etc/kubernetes/admin.conf 파일이 필요합니다. 홈 디렉토리에 복사하고 소유자를 변경한 이후에 **KUBECONFIG** 환경변수에 위치를 지정합니다.

```bash
sudo cp /etc/kubernetes/admin.conf ./k8s-admin.conf
sudo chown vagrant:vagrant k8s-admin.conf 
export KUBECONFIG=/home/vagrant/k8s-admin.conf
echo "export KUBECONFIG=/home/vagrant/k8s-admin.conf" >> .bashrc
kubectl get nodes

NAME    STATUS     ROLES    AGE     VERSION
k8s-1   NotReady   master   8m48s   v1.12.2
k8s-2   NotReady   <none>   2m31s   v1.12.2
k8s-3   NotReady   <none>   2m28s   v1.12.2
```



## Install CNI

**kubectl get nodes** 명령 결과를 보면 **STATUS** 가 현재 **NotReady** 입니다. 초기화 단계에서 선택한 CNI 를 설치해야 실제로 사용 가능한 상태가 됩니다.

**Calico** CNI 를 사용하기 위해 **kubectl** 명령어를 이용해 설치합니다. 

```bash
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml

clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
```

```bash
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

configmap/calico-config created
service/calico-typha created
deployment.apps/calico-typha created
poddisruptionbudget.policy/calico-typha created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
```

```bash
kubectl get nodes
NAME    STATUS   ROLES    AGE   VERSION
k8s-1   Ready    master   25m   v1.12.2
k8s-2   Ready    <none>   19m   v1.12.2
k8s-3   Ready    <none>   19m   v1.12.2
```



## Master isolation

Kubernetes 기본 설정은 Master 역할을 하는 노드에 다른 컨테이너를 배포하지 않도록 되어있습니다. 실습을 진행할 때는 Master 노드도 사용하기 위해 설정을 변경합니다.

```bash
kubectl taint nodes --all node-role.kubernetes.io/master- 

node/k8s-1 untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found
```



## Install dashboard

Kubernetes 를 편하게 사용하기 위해 Dashboard 를 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```

Dashboard 에서 사용할 계정을 생성하는데, 여기에서는 관리자 권한을 준 **admin-user** 를 생성하여 접속하는데 이용합니다.

**kubernetes/dashboard-service-account.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

**kubernetes/dashboard-clusterrolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

위 두 파일을 이용하여 Dashboard 에 접속할 때 사용할 계정을 생성합니다.

```bash
kubectl create -f kubernetes/dashboard-service-account.yaml 
serviceaccount/admin-user created
kubectl create -f kubernetes/dashboard-clusterrolebinding.yaml 
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

설치한 Dashboard 상태를 확인합니다.

```bash
kubectl get svc -n kube-system

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
calico-typha           ClusterIP   10.100.9.93     <none>        5473/TCP        2m53s
kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   24m
kubernetes-dashboard   ClusterIP   10.105.107.14   <none>        443/TCP         119s
```

외부에서 접속하기 위해 Dashboard **Service Type** 을 **NodePort** 로 변경합니다.

```bash
kubectl edit svc -n kube-system kubernetes-dashboard
```

vi 에디터 화면에서 **nodePort** 를 추가하고 **type** 에 **NodePort** 를 지정합니다.

```yaml
spec:
  clusterIP: 10.105.107.14
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
    nodePort: 30000
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
```

```bash
$ kubectl get svc -n kube-system kubernetes-dashboard

NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.105.107.14   <none>        443:30000/TCP   3m54s
```

웹 브라우져를 통해 Dashboard 에 접속합니다.

![dashboard-intro](./README.assets/dashboard-intro.png)



## Get dashboard bearer token

Dashboard 에 접속하기 위해 관리자 Token 을 가져옵니다.

```bash
kubectl get secret -n kube-system                                                     

NAME                                             TYPE                                  DATA   AGE
admin-user-token-9m6zn                           kubernetes.io/service-account-token   3      115s
attachdetach-controller-token-htnpk              kubernetes.io/service-account-token   3      5m38s
bootstrap-signer-token-6ztxm                     kubernetes.io/service-account-token   3      5m52s
bootstrap-token-11h5df                           bootstrap.kubernetes.io/token         7      5m52s
calico-node-token-2kxw5                          kubernetes.io/service-account-token   3      2m43s
certificate-controller-token-6lvgq               kubernetes.io/service-account-token   3      5m52s
...
```

```bash
kubectl describe secret admin-user-token-9m6zn -n kube-system                         

Name:         admin-user-token-9m6zn
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 407a5a06-ed68-11e8-a94d-02c44c503abe

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTltNnpuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0MDdhNWEwNi1lZDY4LTExZTgtYTk0ZC0wMmM0NGM1MDNhYmUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.dhPeoOsMCwmvwNFWFPE6Gn16afd0CpY22uOzNliEgYyALoZndU-j2r62gm3W697UzfatWg5Ezj7m52mq3wKkhr1tHZeEUXHBjmRulOh_sbtJJKBOACGDl9yhWSbhb8F5NMfWhqBnpFwKws9uL3mapiN5Pks8z4yky-pZf3SMpFNtvo_FtoynNbnxo_kalOhvMeqNrpZrJZBGCCCFR9Z9uDu3kaDqsVrfNrMZE0Yx6Rk8TIma9_gibSr57va8XSLFa35P31UwFTHiafVFyOSyvp9ZHkVw2Me-V_SYYQmfjZjjBXr8QZSeEjp8mTJMD5R_NInkl37DtVCG6uf8xUuzjw
ca.crt:     1025 bytes
```

마지막 token: 밑에 있는 문자열을 이용해 Dashboard 에 접속할 수 있습니다.

![dashboard-overview](./README.assets/dashboard-overview.png)



# Setup Ceph with Rook

## Ceph and Rook

**Kubernetes** 위에서 동작하는 어플리케이션이 저장 공간을 필요로 할 경우에는 **Persistent Volume** 을 생성하여 연결해주어야 합니다. 여기에서는 **Ceph** 분산 파일 시스템을 이용하여 실행된 노드에 관계없이 원하는 저장 공간을 생성하고 연결하는 데 활용합니다.

Ceph 클러스터를 직접 구축하고 설정하는 것은 쉽지 않은 일이지만, **Rook** 을 이용해 상대적으로 쉽고 편리하게 구축할 수 있습니다. 아래는 Rook 프로젝트 홈페이지에서 가져온 Rook 소개글입니다.

> Rook is an open source **cloud-native storage orchestrator**,
> providing the platform, framework, and support for a diverse set of 
> storage solutions to natively integrate with cloud-native environments.

Rook 을 이용해 클러스터를 생성할 때 사용하는 설정 파일은 https://github.com/rook/rook/tree/release-0.8/cluster/examples/kubernetes/ceph 경로에 있는 것을 사용합니다.



## Initialize Ceph Cluster

VM 노드 3대가 가지고 있는 추가 디스크 중 한 개를(**/dev/sdc**) Ceph 에 할당하여 클러스터를 구성합니다. Ceph  **BlueStore** 는 직접 파티션을 생성하고 관리하기 때문에, 노드에 장착되어 있는 빈 디스크를 직접 지정합니다. 미리 작성해둔 Vagrantfile 에서는 **/dev/sdc**, **/dev/sdc** 장치에 아무런 파티션 작업도 하지 않은 디스크를 연결해두었습니다.

**Rook Operator** 를 통해 Ceph 클러스터를 생성할 때 필요한 몇 가지 설정을 **rook/operator.yaml** 파일에 기록합니다.

**rook/operator.yaml**

```yaml
    ...
    spec:
      serviceAccountName: rook-ceph-system
      containers:
      - name: rook-ceph-operator
        image: rook/ceph:v0.8.3
        ...
        - name: ROOK_ALLOW_MULTIPLE_FILESYSTEMS
          value: "true"
        ...
        - name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED
          value: "true"
        ...
```

**rook/ceph** 컨테이너 버전을 현재 Stable 상태인 **v0.8.3** 으로 지정합니다. **Shared File System** 을 두 개 이상 만들어서 사용하려면 **ROOK_ALLOW_MULTIPLE_FILESYSTEMS** 옵션을 **"true"** 로 지정합니다. 

**BlueStore** 를 사용하려면 컨테이너에서 직접 파일 시스템을 생성할 수 있어야 하기 때문에 **ROOK_HOSTPATH_REQUIRES_PRIVILEGED** 옵션에 **"true"** 를 지정합니다.

**operator.yaml** 파일을 수정하고 **Rook operator** 를 배포합니다.

```bash
kubectl create -f rook/operator.yaml

namespace/rook-ceph-system created
customresourcedefinition.apiextensions.k8s.io/clusters.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/filesystems.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/objectstores.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/pools.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/volumes.rook.io created
clusterrole.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
role.rbac.authorization.k8s.io/rook-ceph-system created
clusterrole.rbac.authorization.k8s.io/rook-ceph-global created
serviceaccount/rook-ceph-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-system created
clusterrolebinding.rbac.authorization.k8s.io/rook-ceph-global created
deployment.apps/rook-ceph-operator created
```

**cluster.yaml** 파일에 Ceph 에서 사용할 디스크 장치명을 나열합니다.

**rook/cluster.yaml**

```yaml
  ...
  storage:
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      storeType: bluestore
      databaseSizeMB: "512"
      journalSizeMB: "512"
    nodes:
      - name: "k8s-1"
        devices:
        - name: "sdc"
      - name: "k8s-2"
        devices:
        - name: "sdc"
      - name: "k8s-3"
        devices:
        - name: "sdc"
```

각 노드에서 사용할 디스크 장치명을 추가한 후 Ceph 클러스터를 생성합니다.

```bash
kubectl create -f rook/cluster.yaml

namespace/rook-ceph created
serviceaccount/rook-ceph-cluster created
role.rbac.authorization.k8s.io/rook-ceph-cluster created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster created
cluster.ceph.rook.io/rook-ceph created
```

만약에 Rook 으로 Ceph 클러스터를 한번 생성한 이후에 삭제하고 다시 생성하려면 **/var/lib/rook/osd-\***, **/var/lib/rook/mon-\*** 디렉토리를 모두 지운 이후에 위 명령을 다시 실행합니다.

디스크 파티션 작업을 마무리하고 모두 완료되면 **rook-ceph** namespace 에서 배포된 Ceph 구성요소를 확인할 수 있습니다.

```bash
kubectl get deploy -n rook-ceph

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rook-ceph-mgr-a   1         1         1            1           112s
rook-ceph-mon-a   1         1         1            1           2m31s
rook-ceph-mon-b   1         1         1            1           2m15s
rook-ceph-mon-c   1         1         1            1           2m5s
rook-ceph-osd-0   1         1         1            1           95s
rook-ceph-osd-1   1         1         1            1           94s
rook-ceph-osd-2   1         1         1            1           93s
```



## Ceph Dashboard

Ceph Dashboard 를 외부에서 접속할 수 있게 해주는 Service 객체를 생성합니다.

**rook/dashboard-external-http.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-mgr-dashboard-external-http
  namespace: rook-ceph
  labels:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
spec:
  ports:
  - name: dashboard
    port: 7000
    protocol: TCP
    targetPort: 7000
    nodePort: 30001
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  sessionAffinity: None
  type: NodePort
```

외부에서 Dashboard 에 접속할 수 있게 NodePort 를 할당합니다.

```bash
kubectl create -f rook/dashboard-external-http.yaml

service/rook-ceph-mgr-dashboard-external-http created
```

생성한 Service 객체를 확인합니다.

```bash
kubectl get svc -n rook-ceph

NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
rook-ceph-mgr                           ClusterIP   10.103.51.139    <none>        9283/TCP         7m21s
rook-ceph-mgr-dashboard                 ClusterIP   10.96.111.38     <none>        7000/TCP         7m20s
rook-ceph-mgr-dashboard-external-http   NodePort    10.99.153.69     <none>        7000:30001/TCP   37s
rook-ceph-mon0                          ClusterIP   10.100.187.206   <none>        6790/TCP         7m46s
rook-ceph-mon1                          ClusterIP   10.100.234.23    <none>        6790/TCP         7m36s
```

클러스터만 선언한 Ceph 상태를 보여주는 화면입니다.

![ceph-dashboard](./README.assets/ceph-dashboard.png)

동작 중인 OSD Daemon 목록과 각 OSD 별 용량을 볼 수 있습니다.

![ceph-osd](./README.assets/ceph-osd.png)



## Ceph Storages provide by Rook

Rook 을 통해서 Ceph 의 세 가지 저장소를 Kubernetes 에서 사용할 수 있습니다.

| Type               | Description                                         | Access          |
| ------------------ | --------------------------------------------------- | --------------- |
| Block Storage      | 하나의 Pod 만 사용할 수 있는 저장소                 | Inside          |
| Shared File System | 여러 Pod 이 공유할 수 있는 저장소                   | Inside          |
| Object Storage     | S3 API 를 통해 어플리케이션이 사용할 수 있는 저장소 | Inside, Outside |



## Provison Storage

Ceph 저장소를 Kubernetes 에서 사용하기 위해 **StorageClass** 를 등록합니다. Kubernetes 는 **PersistentVolume** 을 생성할 때 미리 등록해둔 StorageClass 를 이용해 Ceph 저장소를 생성합니다.

**rook/storageclass.yaml**

```yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
  fstype: xfs
```

위에서 지정한 **Replicapool** 은 두 개의 복제본을 유지합니다.

```bash
kubectl create -f rook/storageclass.yaml

pool.ceph.rook.io/replicapool created
storageclass.storage.k8s.io/rook-ceph-block created
```

생성한 StorageClass 는 이후에 어플리케이션을 독립적인 **Persistent Volume** 에 붙여서 배포할 때 사용합니다.



## Create Shared File System

여러 Pod 이 동시에 공유해서 사용할 파일 시스템을 생성합니다. **rook/operator.yaml** 파일 속성 중에 **ROOK_ALLOW_MULTIPLE_FILESYSTEMS** 값을 **"true"** 로 지정하였다면, 두 개 이상의 파일 시스템을 생성해서 사용할 수 있습니다.

**rook/filesystem.yaml**

```yaml
apiVersion: ceph.rook.io/v1beta1
kind: Filesystem
metadata:
  name: k8s-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 2
  dataPools:
    - failureDomain: osd
      replicated:
        size: 2
  metadataServer:
    activeCount: 1
    activeStandby: true
    placement:
    resources:
```

**k8s-fs** 이름을 가진 **File System** 을 생성합니다.

```bash
kubectl create -f rook/filesystem.yaml

filesystem.ceph.rook.io/k8s-fs created
```

생성한 File System 을 Ceph Dashboard 화면에서 확인할 수 있습니다.

![ceph-filesystem](./README.assets/ceph-filesystem.png)

파일 시스템 상세 페이지에서 세부적인 내용을 살펴볼 수 있습니다.

![ceph-filesystem-2](./README.assets/ceph-filesystem-2.png)



## Extend Ceph Storage

Ceph 클러스터 디스크 공간을 확장해보겠습니다. 앞에서 Ceph 클러스터를 생성할 때 사용한 **cluster.yaml** 파일을 열어 각 노드에서 사용할 디스크 장치를 추가합니다.

**rook/cluster.yaml**

```yaml
 ...
  storage:
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      storeType: bluestore
      databaseSizeMB: "512"
      journalSizeMB: "512"
    nodes:
      - name: "k8s-1"
        devices:
        - name: "sdc"
        - name: "sdd"
      - name: "k8s-2"
        devices:
        - name: "sdc"
        - name: "sdd"
      - name: "k8s-3"
        devices:
        - name: "sdc"
        - name: "sdd"
```

Rook 을 통해 Ceph 클러스터 공간을 확장합니다.

```bash
kubectl apply -f rook/cluster.yaml

namespace/rook-ceph unchanged
serviceaccount/rook-ceph-cluster unchanged
role.rbac.authorization.k8s.io/rook-ceph-cluster unchanged
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt unchanged
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster unchanged
cluster.ceph.rook.io/rook-ceph configured
```

Ceph Dashboard 를 통해 공간이 늘어난 것을 확인할 수 있습니다.

![ceph-extend](./README.assets/ceph-extend.png)



## Rook Toolbox

Rook **Toolbox** 는 Ceph 를 관리하는 데 필요한 다양한 도구들을 미리 설치해둔 컨테이너입니다. Toolbox 를 이용해 Ceph 클러스터 설정을 직접 살펴보고 변경할 수 있습니다.

Rook Toolbox 를 Kubernetes 에 배포합니다.

```bash
kubectl create -f rook/toolbox.yaml

deployment.apps/rook-ceph-tools created
```

Toolbox 이미지를 받아서 실행되는지 확인합니다.

```bash
kubectl -n rook-ceph get pods -w

NAME                                    READY   STATUS      RESTARTS   AGE
rook-ceph-mds-k8s-fs-5f87c5fb64-57qzn   1/1     Running     0          51m
rook-ceph-mds-k8s-fs-5f87c5fb64-btftv   1/1     Running     0          51m
rook-ceph-mgr-a-5f6dd98574-x5kgf        1/1     Running     0          74m
rook-ceph-mon0-jwxdk                    1/1     Running     0          18m
rook-ceph-mon1-tdlcr                    1/1     Running     0          21m
rook-ceph-osd-id-0-6bd66fc767-jwtk8     1/1     Running     0          74m
rook-ceph-osd-id-1-5bc5c8459d-fjwhv     1/1     Running     0          74m
rook-ceph-osd-id-2-68469f46bb-ln4m7     1/1     Running     0          74m
rook-ceph-osd-id-3-7777db5b7c-sbt5q     1/1     Running     0          7m40s
rook-ceph-osd-id-4-69b8fc69b6-dq5ks     1/1     Running     0          7m38s
rook-ceph-osd-id-5-867f946f7f-h4mxm     1/1     Running     0          7m35s
rook-ceph-osd-prepare-k8s-1-w8ptv       0/1     Completed   0          7m56s
rook-ceph-osd-prepare-k8s-2-86js9       0/1     Completed   0          7m54s
rook-ceph-osd-prepare-k8s-3-hd2tv       0/1     Completed   0          7m52s
rook-ceph-tools-5bc8b8f97-d2h4n         1/1     Running     0          51s
```

Toolbox 가 실행되는 것을 확인한 후에 컨테이너에 접속합니다.

```bash
kubectl -n rook-ceph exec -it rook-ceph-tools-5bc8b8f97-d2h4n bash
```

ceph, rados 등의 명령어를 이용해 Ceph 클러스터 설정을 확인하고 변경할 수 있습니다.

```bash
[root@k8s-2 /]# ceph status
  cluster:
    id:     af422e45-a3a0-488a-9888-50823aac830d
    health: HEALTH_WARN
            clock skew detected on mon.rook-ceph-mon1

  services:
    mon: 2 daemons, quorum rook-ceph-mon0,rook-ceph-mon1
    mgr: a(active)
    mds: k8s-fs-1/1/1 up  {0=k8s-fs-5f87c5fb64-btftv=up:active}, 1 up:standby-replay
    osd: 6 osds: 6 up, 6 in

  data:
    pools:   3 pools, 300 pgs
    objects: 21 objects, 2246 bytes
    usage:   6181 MB used, 48724 MB / 54905 MB avail
    pgs:     300 active+clean

  io:
    client:   852 B/s rd, 2 op/s rd, 0 op/s wr
```

```bash
[root@k8s-2 /]# ceph osd status

+----+-------------------------------------+-------+-------+--------+---------+--------+---------+-----------+
| id |                 host                |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+-------------------------------------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | rook-ceph-osd-id-0-6bd66fc767-jwtk8 | 1030M | 8120M |    0   |     0   |    0   |     0   | exists,up |
| 1  | rook-ceph-osd-id-1-5bc5c8459d-fjwhv | 1030M | 8120M |    0   |     0   |    1   |    90   | exists,up |
| 2  | rook-ceph-osd-id-2-68469f46bb-ln4m7 | 1030M | 8120M |    0   |     0   |    0   |     0   | exists,up |
| 3  | rook-ceph-osd-id-3-7777db5b7c-sbt5q | 1030M | 8120M |    0   |     0   |    0   |     0   | exists,up |
| 4  | rook-ceph-osd-id-4-69b8fc69b6-dq5ks | 1030M | 8120M |    0   |     0   |    0   |     0   | exists,up |
| 5  | rook-ceph-osd-id-5-867f946f7f-h4mxm | 1030M | 8120M |    0   |     0   |    1   |    16   | exists,up |
+----+-------------------------------------+-------+-------+--------+---------+--------+---------+-----------+
```

```bash
[root@k8s-2 /]# ceph df

GLOBAL:
    SIZE       AVAIL      RAW USED     %RAW USED
    54905M     48724M        6181M         11.26
POOLS:
    NAME                ID     USED     %USED     MAX AVAIL     OBJECTS
    k8s-fs-metadata     1      2246         0        22988M          21
    k8s-fs-data0        2         0         0        22988M           0
    replicapool         3         0         0        22988M           0
```

```bash
[root@k8s-2 /]# rados df

POOL_NAME          USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS      RD WR_OPS    WR
k8s-fs-data0        0 B       0      0      0                  0       0        0      0     0 B      0   0 B
k8s-fs-metadata 2.2 KiB      21      0     42                  0       0        0   1190 595 KiB     44 8 KiB
replicapool         0 B       0      0      0                  0       0        0      0     0 B      0   0 B

total_objects    21
total_used       6.0 GiB
total_avail      48 GiB
total_space      54 GiB
```



## Change Ceph configuration through Toolbox

Toolbox 컨테이너에 접속하여 현재 설정되어 있는 pg_num 값을 확인합니다.

```bash
kubectl -n rook-ceph exec -it rook-ceph-tools-5bc8b8f97-d2h4n bash

[root@k8s-2 /]# ceph osd lspools
1 k8s-fs-metadata,2 k8s-fs-data0,3 replicapool,

[root@k8s-2 /]# ceph osd pool get k8s-fs-data0 pg_num
pg_num: 100
```

`ceph osd pool set` 명령을 이용해 pg_num 값을 150으로 변경합니다.

```bash
[root@k8s-2 /]# ceph osd pool set k8s-fs-data0 pg_num 150
set pool 2 pg_num to 150

[root@k8s-2 /]# ceph osd pool set replicapool pg_num 150
set pool 3 pg_num to 150
```

Dashboard 화면에서 변경된 값을 확인할 수 있습니다.

![ceph-changed-status](./README.assets/ceph-changed-status.png)



## Remove Ceph node

현재 세 대로 구성되어 있는 Ceph 클러스터에서 한 노드를 제거해보겠습니다. cluster.yaml 를 수정하여 삭제하려는 노드와 관련된 내용을 제거합니다.

**rook/cluster.yaml**

```yaml
 ...
  storage:
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      storeType: bluestore
      databaseSizeMB: "512"
      journalSizeMB: "512"
    nodes:
      - name: "k8s-2"
        devices:
        - name: "sdc"
        - name: "sdd"
      - name: "k8s-3"
        devices:
        - name: "sdc"
        - name: "sdd"
```

Rook Operator 를 이용해 실제 Ceph 클러스터에 적용합니다.

```bash
kubectl apply -f rook/cluster.yaml

namespace/rook-ceph unchanged
serviceaccount/rook-ceph-cluster unchanged
role.rbac.authorization.k8s.io/rook-ceph-cluster unchanged
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt unchanged
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster unchanged
cluster.ceph.rook.io/rook-ceph configured
```

Ceph Dashboard 를 통해 노드가 삭제된 것을 확인할 수 있습니다.

![ceph-reduced](./README.assets/ceph-reduced.png)

OSD 데몬 화면에서 2개가 줄어들어 4개만 남아있는 것을 볼 수 있습니다.

![ceph-reduced-osd](./README.assets/ceph-reduced-osd.png)



# Deploy Application with Ceph

## Deploy Minio

Minio 어플리케이션을 Shared File System 과 함께 배포해보겠습니다.

배포하기 전에 File System 안에 사용할 디렉토리를 먼저 만드는 것이 필요합니다. 여기에서는 간단하게 nginx 컨테이너 내부 /tmp/fs 디렉토리에 Share File System 을 붙인 후에 디렉토리를 생성합니다.

```bash
kubectl create -f rook/nginx-fs-deploy.yaml

deployment.apps/nginx-fs created
```

실행된 Pod 이름을 확인합니다.

```bash
kubectl get pod

NAME                        READY   STATUS    RESTARTS   AGE
nginx-fs-5bfc8dbf5f-5ggz8   1/1     Running   0          77s
```

**kubectl exec** 명령을 이용해 앞에서 실행한 Pod 에 접속해 필요한 디렉토리를 미리 생성해둡니다.

```bash
rook$ kubectl exec -it nginx-fs-5bfc8dbf5f-5ggz8 /bin/bash
root@nginx-fs-5bfc8dbf5f-5ggz8:/# cd /tmp/fs
root@nginx-fs-5bfc8dbf5f-5ggz8:/tmp/fs# mkdir minio
root@nginx-fs-5bfc8dbf5f-5ggz8:/tmp/fs# exit
exit
```



**minio/minio-deploy.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: minio
spec:
  template:
    metadata:
      labels:
        k8s-app: minio
    spec:
      containers:
      - name: minio
        volumeMounts:
        - name: minio-store
          mountPath: "/data"
        image: minio/minio
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
      volumes:
      - name: minio-store
        flexVolume:
          driver: ceph.rook.io/rook
          fsType: ceph
          options:
            fsName: k8s-fs
            clusterNamespace: rook-ceph
            path: /minio
```

minio 를 클러스터에 배포합니다.

```bash
kubectl create -f minio/minio-deploy.yaml

deployment.extensions/minio created
```

```bash
kubectl create -f minio/minio-svc.yaml

service/minio-svc created
```



배포한 minio 저장소에 파일을 저장해보겠습니다.

![minio](D:\Dropbox\Works\Ceph-201812-Meetup\README.assets\minio.png)



## Deploy MySQL

MySQL 어플리케이션을 **Block Storage** 와 함께 배포해보겠습니다.

먼저 앞에서 생성한 **StorageClass** 이름으로 **PersistentVolumeClaim** 을 생성합니다.



**mysql/mysql-pvc.yaml**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    k8s-app: mysql
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl create -f mysql/mysql-pvc.yaml

persistentvolumeclaim/mysql-pvc created
```

MySQL 를 배포할 때 컨테이너에 앞에서 생성한 mysql-pvc 를 붙여줍니다.



**mysql/mysql-deploy.yaml**

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mysql
  labels:
    k8s-app: mysql
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        k8s-app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: changeme
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

```bash
kubectl create -f mysql/mysql-deploy.yaml

deployment.apps/mysql created
```

```bash
kubectl create -f mysql/mysql-svc.yaml

service/mysql created
```



생성한 MySQL 서버에 접속하여 제대로 동작하고 있는지 확인해봅니다.

```bash
kubectl get svc

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          149m
minio-svc    NodePort    10.111.32.46    <none>        9000:30010/TCP   4m54s
mysql        NodePort    10.102.219.47   <none>        3306:30020/TCP   42s
```

서비스 내용을 확인하고 mysql client 를 이용해 접속합니다.

```bash
mysql -uroot -p -h 10.254.1.2 -P 30020

Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)
```



## Deploy Ghost

Ghost 어플리케이션에서 사용할 디렉토리를 Share File System 에 미리 생성해둡니다.

```bash
kubectl exec -it nginx-fs-5bfc8dbf5f-5ggz8 /bin/bash

root@nginx-fs-5bfc8dbf5f-5ggz8:/# cd /tmp/fs
root@nginx-fs-5bfc8dbf5f-5ggz8:/tmp/fs# mkdir ghost
root@nginx-fs-5bfc8dbf5f-5ggz8:/tmp/fs# ls -al
total 4
drwxr-xr-x 1 root root    2 Nov 18 15:35 .
drwxrwxrwt 1 root root 4096 Nov 18 15:05 ..
drwxr-xr-x 1 root root    0 Nov 18 15:35 ghost
drwxr-xr-x 1 root root    2 Nov 18 15:15 minio
```

MySQL 에 접속하여 사용할 데이터베이스를 생성합니다.

```bash
mysql -uroot -p -h 10.254.1.2 -P 30020

Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database ghost;
Query OK, 1 row affected (0.02 sec)
```



PersistentVolume 과 Database 생성을 완료한 후에 Ghost 어플리케이션을 배포합니다.

**ghost/ghost-deploy.yaml**

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: ghost
spec:
  template:
    metadata:
      labels:
        k8s-app: ghost
    spec:
      containers:
      - name: ghost
        volumeMounts:
        - name: ghost-volume
          mountPath: "/var/lib/ghost/content"
        image: ghost:2
        env:
        - name: database__client
          value: "mysql"
        - name: database__connection__host
          value: "mysql"
        - name: database__connection__user
          value: "root"
        - name: database__connection__database
          value: "ghost"
        - name: database__connection__password
          value: "changeme"
        ports:
        - containerPort: 2368
      volumes:
      - name: ghost-volume
        flexVolume:
          driver: ceph.rook.io/rook
          fsType: ceph
          options:
            fsName: k8s-fs
            clusterNamespace: rook-ceph
            path: /ghost
```

Ghost 어플리케이션을 배포합니다.

```bash
kubectl create -f ghost/ghost-deploy.yaml

deployment.apps/ghost created
```

외부에서 접속할 수 있도록 Service 를 생성합니다.

```bash
kubectl create -f ghost/ghost-svc.yaml

service/ghost-svc created
```

Ghost 어플리케이션이 배포된 것을 확인합니다.

```bash
kubectl get deploy ghost

NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
ghost   1         1         1            1           6m9s
```



배포가 완료된 후 웹 페이지에 접속하여 확인합니다.

![ghost](./assets/ghost.png)

MySQL 에 테이블이 제대로 생성되었는지 확인해봅니다.

```bash
mysql -uroot -p -h 10.254.1.2 -P 30020

Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 27
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use ghost;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------------+
| Tables_in_ghost        |
+------------------------+
| accesstokens           |
| api_keys               |
| app_fields             |
| app_settings           |
| apps                   |
| brute                  |
| client_trusted_domains |
| clients                |
| integrations           |
| invites                |
| migrations             |
| migrations_lock        |
| mobiledoc_revisions    |
| permissions            |
| permissions_apps       |
| permissions_roles      |
| permissions_users      |
| posts                  |
| posts_authors          |
| posts_tags             |
| refreshtokens          |
| roles                  |
| roles_users            |
| sessions               |
| settings               |
| subscribers            |
| tags                   |
| users                  |
| webhooks               |
+------------------------+
29 rows in set (0.00 sec)
```



# Summary

**Kubernetes** 클러스터를 이용해 어플리케이션을 배포할 때 **Ceph** 를 **Persistent Storage** 로 활용하는 실습을 진행하였습니다. **HostPath** Volume 을 이용할 경우에는 어플리케이션이 특정한 노드에서 동작해야 하기 때문에 Kubernetes 클러스터를 적극 활용하는데 제약을 받을 수 있습니다. Ceph 의 **Shared File System** 을 이용해 여러 컨테이너에서 공유할 공간으로 사용하거나, **Block Storage** 를 생성해 특정 컨테이너를 위한 전용 저장 공간을 붙여  사용할 수도 있습니다.

실습 내용에 대하여 부족하거나 보완해야할 점이 있다면 있다면 메일로 보내주세요. 감사합니다.

임찬식 (chanshik@gmail.com) 