---
title: "Kubernetes v1.30でBetaになったUser Namespaceで遊んでみました"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "UserNamespace"]
published: true
---

# はじめに
Kubernetes v1.30でBetaになったuser namespaceをFedora 40, CentOS Stream 9, Ubuntu 24.04で試してみました。[gistに書いたメモ](https://gist.github.com/orimanabu/8fc59eccd9c8c6787314e8dabc03df8d)の改訂版です。

- 参考資料: [Kubernetes 1.30: Beta Support For Pods With User Namespaces](https://kubernetes.io/blog/2024/04/22/userns-beta/)


下記のドキュメントに沿って環境を作って検証しました。CRI-O、crunを入れて、`/etc/sub[ug]id` をちゃんと設定すれば、たぶんすんなり動くと思います。

- [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/)
- [Use a User Namespace With a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)

本記事は、2024-04-27時点の情報をもとにしています。

現時点では、user namespaceを試すには下記が必要になります。

- Linuxカーネル v6.3 (tmpfsのidmapマウント)
  - CentOS Stream 9はこれよりも古いカーネルだが、必要な機能がバックポートされていると思われる (後述)
- crun v1.9以降 (v1.13以降が推奨)
  - runc v1.1.zは現時点では未対応
- CRI-O v1.25以降
  - containerd v1.7は、k8s v1.27〜v1.30のuser namespaceには未対応

# セットアップ

## VMの作成
KVMホスト上に、仮想マシンを必要なだけ作ります。
以下の検証では、作った順番の都合で、結果的に

- control plane: Fedora 40
- worker: Fedora 40, CentOS Stream 9, Ubuntu 24.04

という構成になりました。

## いろいろ準備

### swap無効化

#### Fedora 40の場合:

```
sudo touch /etc/systemd/zram-generator.conf
```

#### CentOS Stream 9、Ubuntu 24.04の場合

/etc/fstabのswapの行をコメントアウトして、`swapoff -a` を実行します。

### Firewalldの設定

Fedora 40、CentOS Stream 9の場合のみ実施します。Ubuntu 24.04の場合はこの手順はスキップしてください。

#### Kubernetesで必要なポートの開放

```
sudo firewall-cmd --add-port 6443/tcp --permanent
sudo firewall-cmd --add-port 10250/tcp --permanent
sudo firewall-cmd --reload
```

#### Antreaで使用するポートの開放

後述のとおり、CNIプラグインとしてAntreaを使用しました。必要と思われるポートをあらかじめ開けておきます。

```
sudo firewall-cmd --permanent --add-port 10349/tcp
sudo firewall-cmd --permanent --add-port 10350/tcp
sudo firewall-cmd --permanent --add-port 10351/tcp
sudo firewall-cmd --permanent --add-port 10351/udp
sudo firewall-cmd --permanent --add-port 6081/udp
sudo firewall-cmd --permanent --add-port 443/tcp
sudo firewall-cmd --reload
```

### `net.ipv4.ip_forward` の設定

```
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/98-k8s.conf
sudo sysctl -p --system
```

### `/run/systemd/resolve/resolv.conf` の作成

CentOS Stream 9は`systemd-resolved` を使っていないせいか、このファイルがないためにkubeadm joinに失敗します。
正しい対応がよくわかっていませんが、とりあえず下記を実行しておくとうまくいきました。

```
sudo mkdir -p /run/systemd/resolve
sudo ln -s /etc/resolv.conf /run/systemd/resolve/resolv.conf
```

### `kubelet` ユーザーのuid/gid mappingの設定

(※ 検証をする上で、この手順は必ずしも実施しなくても構いません)

ユーザー `kubelet` を作成し、[ここ](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/#set-up-a-node-to-support-user-namespaces)
に記載されているように `65536 * 110` 個のUIDを使えるようにします。

```
sudo useradd kubelet
```

デフォルト設定だと、使えるUIDは65536個です (下記出力例はFedora 40です。`ori` はインストール時に作成したユーザー名です)。

```
$ cat /etc/subuid
ori:524288:65536
kubelet:589824:65536
```

`kubelet` ユーザーが使えるUIDを `65536 * 110` 個に増やしておきます。マッピングの開始番号(sub[gu]idの2つ目のカラム、上記例だと `589824` のところ)が65536の倍数である必要がありますが、CentOS Stream 9、Ubuntu 24.04の場合は、そうなっていない可能性があります。必要に応じて、開始番号も調整してください。本検証では、Fedora 40で作ったときの番号をコピペして、CentOS Stream 9およびUbuntu 24.04の場合も下記の値を使用しました。

```
$ cat /etc/subuid
ori:524288:65536
kubelet:589824:7208960
```

## Kubernetesのインストール

[ここ](https://github.com/cri-o/packaging?tab=readme-ov-file#distributions-using-rpm-packages)の情報にしたがってKubernetesおよびCRI-Oのrpmをインストールします。
CRI-Oはk8sと同じバージョンを入れる必要がありますが、v1.30のCRI-Oはまだ正式には出ていないため、開発中のものを使います。

### Fedora 40、CentOS Stream 9の場合

k8sのyumリポジトリを設定します。

```
KUBERNETES_VERSION=v1.30
```

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF
```

パッケージをインストールします。

```
sudo dnf install -y kubelet kubeadm kubectl
```

サービスを起動設定をします(サービスの起動は `kubeadm init` の中でやってくれます)。

```
sudo systemctl enable kubelet
```

## CRI-Oのインストール

yumリポジトリを設定します。

```
PROJECT_PATH=prerelease:/main
```

```
cat <<EOF | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/rpm/repodata/repomd.xml.key
EOF
```

パッケージをインストールします。

```
sudo dnf install -y cri-o
```

もしここで `error: Verifying a signature using certificate DE15B14486CD377B9E876E1A234654DA9A296436 (isv:kubernetes OBS Project <isv:kubernetes@build.opensuse.org>)` みたいなエラーが出たら、
`/var/cache` 以下にダウンロードされているrpmファイルを直接インストールして乗り切ります (Fedoraで起こる気がします、が原因は追いかけてません...)。具体的には、 `sudo find /var/cache -name 'cri-o.*.rpm'` して出てきたrpmファイルを `sudo rpm -ivh --nosignature` で入れます。

CRI-Oが必要とするコマンドがいくつかあるので、それをインストールします。雑にやるならpodmanを入れると依存関係で入れてくれます(podman自体はCRI-Oを使用するのに必要はありませんが）。

```
sudo dnf install -y podman
```

もし真面目に必要なものだけ入れるなら、これくらいを入れておけば大丈夫だと思います。

```
sudo dnf install container-selinux crun conmon containers-common shadow-utils-subid iproute-tc
```

サービスを起動します。

```
sudo systemctl start crio
sudo systemctl enable crio
```

### Ubuntu 24.04の場合

[ここ](https://github.com/cri-o/packaging?tab=readme-ov-file#distributions-using-deb-packages)のとおりにやりました。

まず前提となるパッケージを入れます。

```
sudo apt-get update
sudo apt-get install -y software-properties-common curl
```

次にリポジトリの設定をします。[24.04から形式が変わった](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0810#sec6)ようですが、混在できるようなので古いままでやりました。

```
# k8s repo
KUBERNETES_VERSION=v1.30
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# crio repo
PROJECT_PATH=prerelease:/main
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/$PROJECT_PATH/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
```

k8s, cri-oを入れます。

```
sudo apt-get update
sudo apt-get install -y cri-o kubelet kubeadm kubectl podman
```

crunやその他依存関係で必要そうなパッケージもインストールしたいので、podmanもインストールしています (podman自体は直接必要なわけではありません)。

最後に、サービスの起動設定をしておきます。

```
sudo systemctl enable kubelet
sudo systemctl enable crio
sudo systemctl start crio
```

## kubeadmの実行

`UserNamespacesSupport` のfeature gateを有効にしたconfigファイルを作ります。

```
cat <<EOF > kubeadm.config
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    feature-gates: "UserNamespacesSupport=true"
networking:
  serviceSubnet: 10.96.0.0/24
  podSubnet: 10.244.0.0/16
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  UserNamespacesSupport: true
EOF
```

control planeノードでkubeadmを実行します。

```
sudo kubeadm init --config=kubeadm.config
```

正常終了したら、画面の出力どおりにkubeconfigをホームディレクトリにコピーして、workerノード上で `kubeadm join` を実行します。

最後に好きなCNIプラグインを入れます。この検証では、Antreaを入れました。

```
kubectl apply -f https://github.com/antrea-io/antrea/releases/download/v1.13.4/antrea.yml
```

※ 最初Ciliumで検証していたのですが、Ubuntu 24.04でCiliumがうまく動かなかったため([issue](https://github.com/cilium/cilium/issues/32198))、Antreaに変えました。

# user namespaceの検証

[ここ](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/#create-pod)のサンプルを使います。

```
cat <<EOF > user-namespaces-stateless.yaml
apiVersion: v1
kind: Pod
metadata:
  name: testpod
spec:
  hostUsers: false
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
```

`spec.hostUsers` に `false` を設定することで、Pod内のコンテナプロセスがuser namespaceの中で動きます。

※ 余談ですがCRI-Oは、Kubernetesが正式にuser namespaceに対応する前から、独自にuser namespaceを使えるようになっていました (Pod manifestの `metadata.annotations` に `io.kubernetes.cri-o.userns-mode: "auto"` を入れておくとuser namespaceを使います)。Kubernetes側で正式にuser namespaceに対応したので、この拡張は今後はdeprecateされるのではないかと思われます。

デプロイしたらこんな感じになります。

```
$ kubectl get pod -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE                             NOMINATED NODE   READINESS GATES
testpod   1/1     Running   0          24m   10.0.2.208   k8s-fedora-wk2.fedora-amd.home   <none>           <none>
```

コンテナ内では、UID 0でプロセスが動いています。

```
$ kubectl exec testpod -- lsfd -p 1 | head -n 2
COMMAND PID USER  ASSOC MODE TYPE  SOURCE MNTID      INODE NAME
sleep     1 root    exe  ---  REG overlay     0    1255122 /usr/bin/sleep
```

```
$ kubectl exec testpod -- id
uid=0(root) gid=0(root) groups=0(root)
```

`/proc/*/[ug]id_map` を見ると、user namespaceにおけるUIDのマッピングが確認できます。

```
$ kubectl exec pod/testpod -- cat /proc/self/uid_map
         0    4980736      65536
```

コンテナ内のUID 0以降65536個のUIDが、ホスト上のUID 4980736以降の65536個にマッピングされていることがわかります。

※ user namespaceを使用しない場合は、通常 `/proc/*/[ug]id_map` の2つ目のカラムが `0` になります

最後に該当workerノード (`k8s-fedora-wk2`) 上で、このsleepプロセスをpsしてみます。まずホスト上でのPIDを調べます。

```
ssh k8s-fedora-wk2 systemd-cgls -u kubepods-besteffort-pod$(kubectl get pod testpod -o json | jq -r .metadata.uid | tr '-' '_').slice
Unit kubepods-besteffort-pode33ed7f7_0c89_42a5_9956_94a0a1761b23.slice (/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pode33ed7f7_0c89_42a5_9956_94a0a1761b23.slice):
└─crio-7089c8a0f865cf3857f0e4bf493daaee2716607661abbb50a090473ae6ceecbc.scope …
  └─container
    └─9247 sleep infinity
```

コンテナ内のsleepプロセスのホスト上のPIDは9247であることがわかります。

```
$ ssh k8s-fedora-wk2 ps u -p 9247
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
4980736     9247  0.0  0.0   2484  1156 ?        Ss   15:47   0:00 sleep infinity
```

PID 9247のsleepは/proc/self/uid_map` で確認したUID 4980736で動いていることが確認できました。

※ user namespaceを使用しない場合は、ここのPIDがコンテナプロセスのPIDと同じ (このコンテナの場合は `0`) になります。