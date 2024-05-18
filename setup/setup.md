# 構築

## subnet

Node 用サブネット

```bash
10.0.0.0/24
n1: 10.0.0.1/24 192.168.10.11
n2: 10.0.0.2/24 192.168.10.11
n3: 10.0.0.3/24 192.168.10.11
```

## /etc/hosts

hosts file で各ノードを簡単に指定できるようにします。

```bash
cat << _EOF_ > /etc/hosts
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.0.0.1 n1
10.0.0.2 n2
10.0.0.3 n3
_EOF_
```

## containerd

- 全てのノードで実行します。

container runtime をインストールします。
containerd を利用します。

<https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/>

<https://github.com/containerd/containerd/blob/main/docs/getting-started.md>

### repository の追加

```bash
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
dnf -y update
dnf -y install containerd.io
```

### containerd の設定

/etc/containerd/config.toml を編集して runc が systemd cgroup ドライバーを使うようにします。

```bash
containerd config default > /etc/containerd/config.toml
perl -pi -e "s|(SystemdCgroup =).*|\1 true|" /etc/containerd/config.toml
perl -pi -e "s|(sandbox_image = \"registry.k8s.io/pause).*\"|\1:3.9\"|" /etc/containerd/config.toml
```

### 実行

containerd デーモンを立ち上げます。

```bash
systemctl enable containerd.service
systemctl restart containerd.service
```

## swap off

- 全てのノードで実行します。
  kubelet を正常に動作させるために、swap を off にします。

### /etc/fstab

/etc/fstab の swap の行を削除します。

```bash
vi /etc/fstab
```

### systemd.swap

systemd 管理の swap を無効化します。

#### swap の情報

```bash
# systemctl --type swap
  UNIT           LOAD   ACTIVE SUB    DESCRIPTION
  dev-zram0.swap loaded active active Compressed Swap on /dev/zram0
```

#### 無効化

```bash
systemctl mask dev-zram0.swap
reboot
```

## kernel module と kernel paramater の設定

- 全てのノードで実行します。

### kernel module の設定

ipv4 forwarding の有効化 と iptabeles からブリッジされたトラフィックを見えるようにします。

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### kernel paramater の設定

ipv4 forwarding の有効化 と iptabeles からブリッジされたトラフィックを見えるようにします。

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

### 適用

```bash
sudo sysctl --system
```

### 確認

適用できているかの確認をします。

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## kubernetes tools install

- 全てのノードで実行します。

### repository の追加

kubelet kubeadm kubectl などのインストールのためのレポジトリを追加します。

```bash
cat << '_EOF_' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
_EOF_
```

### kubelet kubeadm kubectl の versionlock

各ツールのバージョンを固定します。

#### dnf plugin

dnf の plugin である dnf-plugin-versionlock をインストールします。

```bash
dnf -y install dnf-plugin-versionlock
dnf -y update
```

#### 現在のレポジトリの確認

確認します。

```bash
# rpm -qa |grep kube
kubelet-1.30.0-150500.1.1.x86_64
kubeadm-1.30.0-150500.1.1.x86_64
kubectl-1.30.0-150500.1.1.x86_64
```

#### versionlock

version を固定し各ツールをインストールします。

```bash
dnf versionlock add kubelet kubeadm kubectl
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

## kubelet

- 全てのノードで実行します。

kubelet デーモンを起動します。

```bash
systemctl enable kubelet.service
systemctl restart kubelet.service
```

## クラスタの作成

### kubeadm init

- master node で実行します。

クラスタを立ち上げます。

```bash
kubeadm init \
--kubernetes-version=1.30.0 \
--apiserver-advertise-address=10.0.0.1 \
--pod-network-cidr=10.32.0.0/16 \
--control-plane-endpoint=192.168.10.11
```

#### 各 Linux ユーザーの設定

Linux ユーザーで apiserver と通信するには以下のコマンドを使用します。

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

また、10.0.0.1, 192.168.10.11 と可能なマシンへ scp してもクラスタと通信可能となります。

#### root ユーザーの設定

Linux root ユーザーで apiserver と通信するには以下のコマンドを使用します。

```bash
echo export KUBECONFIG=/etc/kubernetes/admin.conf >> ~/.bashrc
. ~/.bashrc
```

### kubeadm join

- worker node で実行します。

その他 ノード をクラスタに参加させます。

```bash
kubeadm join 192.168.10.11:6443 --token ihs0kf.30cr6ng0q0hu2bkj \
        --discovery-token-ca-cert-hash sha256:bcdb613b9ef4660c7360a7004dac8aff3462f51a1950aa24e42bf354f758dde7
```

### bash_completion, alias

bash_completion, alias の設定を行います。

```bash
cat << '_EOF_' >> ~/.bashrc
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
_EOF_
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bash_aliases
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

### マスターノードでのスケジューリング

マスターノードでも pod がスケジューリングされるようにします。

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### node の確認

- master node で実行します。移行はすべて master node または、その他マシンから行います。
  現在の node の状態を確認します。

```bash
$ kubectl get node  -o wide
NAME   STATUS     ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                                KERNEL-VERSION          CONTAINER-RUNTIME
n1     NotReady   control-plane   19h   v1.30.0   10.0.0.1      <none>        Fedora Linux 39 (Server Edition)        6.8.8-200.fc39.x86_64   containerd://1.6.31
n2     NotReady   <none>          19h   v1.30.0   10.0.0.2      <none>        Fedora Linux 39 (Server Edition)        6.8.8-200.fc39.x86_64   containerd://1.6.31
n3     NotReady   <none>          19h   v1.30.0   10.0.0.3      <none>        Fedora Linux 39 (Server Edition)        6.8.8-200.fc39.x86_64   containerd://1.6.31
```

## calico

overlay network の構築を行います。

### cni plugin の導入

cni を動作させるために必要な plugin を install します。

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-amd64-v1.4.1.tgz
sudo tar -xzvf cni-plugins-linux-amd64-v1.4.1.tgz -C /opt/cni/bin
rm -f cni-plugins-linux-amd64-v1.4.1.tgz
```

#### manifest の適用

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/tigera-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/maeshinshin/kubernetes-cluster-setup/main/setup/custom-resources.yaml
```

containerd,service を再起動します。

```bash
systemctl restart containerd.service
```

#### 確認

```bash
$ kubectl get pods --all-namespaces -o wide
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE   IP             NODE   NOMINATED NODE   READINESS GATES
calico-apiserver   calico-apiserver-6cff856488-w9dd7          1/1     Running   0          12m   10.32.217.2    n2     <none>           <none>
calico-apiserver   calico-apiserver-6cff856488-zzjps          1/1     Running   0          12m   10.32.40.133   n1     <none>           <none>
calico-system      calico-kube-controllers-68d68b5d85-9jtfz   1/1     Running   0          47h   10.32.40.130   n1     <none>           <none>
calico-system      calico-node-242qg                          1/1     Running   0          47h   10.0.0.1       n1     <none>           <none>
calico-system      calico-node-8fptk                          1/1     Running   0          47h   10.0.0.3       n3     <none>           <none>
calico-system      calico-node-znxv6                          1/1     Running   0          47h   10.0.0.2       n2     <none>           <none>
calico-system      calico-typha-568db95676-c22t5              1/1     Running   0          47h   10.0.0.3       n3     <none>           <none>
calico-system      calico-typha-568db95676-xwcj6              1/1     Running   0          47h   10.0.0.2       n2     <none>           <none>
calico-system      csi-node-driver-kr4mz                      2/2     Running   0          47h   10.32.40.129   n1     <none>           <none>
calico-system      csi-node-driver-rlvs9                      2/2     Running   0          47h   10.32.98.1     n3     <none>           <none>
calico-system      csi-node-driver-wvjjf                      2/2     Running   0          47h   10.32.217.1    n2     <none>           <none>
kube-system        coredns-7db6d8ff4d-458dg                   1/1     Running   0          2d    10.32.40.131   n1     <none>           <none>
kube-system        coredns-7db6d8ff4d-bjpt9                   1/1     Running   0          2d    10.32.40.132   n1     <none>           <none>
kube-system        etcd-n1                                    1/1     Running   8          2d    10.0.0.1       n1     <none>           <none>
kube-system        kube-apiserver-n1                          1/1     Running   8          2d    10.0.0.1       n1     <none>           <none>
kube-system        kube-controller-manager-n1                 1/1     Running   0          2d    10.0.0.1       n1     <none>           <none>
kube-system        kube-proxy-52r7s                           1/1     Running   0          2d    10.0.0.2       n2     <none>           <none>
kube-system        kube-proxy-sx675                           1/1     Running   0          2d    10.0.0.3       n3     <none>           <none>
kube-system        kube-proxy-wkclk                           1/1     Running   0          2d    10.0.0.1       n1     <none>           <none>
kube-system        kube-scheduler-n1                          1/1     Running   8          2d    10.0.0.1       n1     <none>           <none>
tigera-operator    tigera-operator-7d5cd7fcc8-g7bcr           1/1     Running   0          47h   10.0.0.3       n3     <none>           <none>
$ kubectl get nodes
NAME   STATUS   ROLES           AGE   VERSION
n1     Ready    control-plane   2d    v1.30.0
n2     Ready    <none>          2d    v1.30.0
n3     Ready    <none>          2d    v1.30.0
```
