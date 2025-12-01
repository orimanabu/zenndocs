---
title: "MetalLBで外部からのBGPの経路をインポートする"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [MetalLB, FRR, BGP, Kubernetes, OpenShift]
published: true
---

# はじめに
外部のBGPルーターから受け取った経路をMetalLBのBGPスピーカーにインポートする話です。本記事はOpenShift Advent Calendar 2025の1日目のエントリーなのでOpenShiftを対象に書いていますが、他のKubernetesディストリビューションでも同じことができると思います。

Kubernetesの `type: LoadBalancer` のService(以下LoadBalancer Service)は、クラウドインフラのロードバランサーAPIと連携して、クラスター外部からL4ロードバランサー経由でクラスター上のPodに接続できるようにする機能です。オンプレミスの環境では、そのような機能は用意することが難しいと思いますが (OpenStack等のオンプレクラウド基盤があればいいのですが...)、そのような機能がインフラ側にない環境でLoadBalancer Serviceを使うための仕組みを提供するのがMetalLBです。MetalLBは2つの動作モードがあります。ひとつはL2モードで、1台のノードにLoadBalancer ServiceのExternal IPを割り当て、External IPのARPリクエストに答えることで外部から接続できるようにします。もうひとつの動作モードがL3モード(BGPモード)で、外部のルーターとBGPピアを張り、External IPのアドレスを/32で広告することで外部から接続できるようにします。本記事では、後者のBGPモードが前提になります。

MetalLBのBGPモードは、開発初期段階ではGo言語による独自のBGP実装を使っていました(私はその時代を知らないのですが...)。その後、BGPの実装に[FRRouting](https://frrouting.org/)(以下FRR)を使うようになり、DaemonSetで動くspeaker PodのサイドカーコンテナとしてFRRを動かすようになりました。そうなると、FRRが持つリッチなルーティング機能を活用したくなるのが人情というものですが、MetalLBの立ち位置はBGPバックボーンの一員となることではなく、LoadBalancer Serviceを広告する "BGP Injector" である、とのことで、受け取った経路をカーネルRIBにインポートしない実装になっていました[^1]。

具体的には、バックエンドとしてFRRを使う場合、FRRのコンフィグテンプレートは外部から学習した経路をdenyするroute-mapが設定されるようになっています。ConfigMap `bgpextras` を使ったカスタムルールで上書きする方法は存在しましたが[^2]、正式サポートな機能ではなく[^3]、ドキュメントにも載っていないため(多分)、おそらく使っている人はほとんどいなかったのではないかと思います。

![](/images/metallb-frr-bgpextras.png)
*ConfigMap `bgpextras` を導入したコミットメッセージ*

その後もFRRを有効活用してBGPベースのネットワークでKubernetesを活用したい人たちの情熱は衰えることなく、MetalLBで使うFRRをspeaker Podのサイドカーコンテナから切り出し、MetalLB以外からもFRRのを使えるようにする実装が出てきました[^4]。具体的には、FRRを切り出してFRR-K8sという仕組みをかぶせ、KubernetesのAPI (カスタムリソースFRRConfiguration) 経由でFRRを利用できるようにします。複数のFRRConfigurationがある場合は、FRR-k8sがいい感じにマージしてFRRのコンフィグを生成します。MetalLBのバックエンドがfrr-k8sの場合は、MetalLBのspeakerがカスタムリソースBGPPeerやBGPAdvertisementを元にFRRConfigurationを生成してFRR-k8sに渡します。

![](/images/metallb-frr-k8s.png)
*MetalLB + FRR-k8sの登場人物*

ここまでが背景です。FRR-k8sの登場によって「複数のFRRConfigurationをマージしてFRRのコンフィグを生成する」ということができることになったのですが、これを応用することでMetalLBで外部から受け取った経路をインポートすることができるようになります。

# セットアップ

具体例を見ていきましょう。まずは、バックエンドとしてfrr-k8sを使うようにMetalLBをセットアップします。Manifest, Kustomize, Helm, Operatorそれぞれのインストール方法でfrr-k8sバックエンドを有効にする方法はupstreamのインストールドキュメント[^5]を参照してください。OpenShiftはv4.17以降だとMetalLB Operatorのデフォルト設定がfrr-k8sになっています。公式サイトの説明にfrr-k8sが **Experimental** だと表現されていますが[^6]、OpenShiftだと本番環境でも使われているので、気にせず試してみましょう。

![](/images/metallb-frr-k8s-experimental.png)
*公式サイトでのFRR-k8sの説明*

図のような環境で検証します[^7]。左下の `r2` というルータの下にOpenShiftのノードがいます。OpenShiftは v4.20.2上にMetalLB Operatorをインストールしました。上述したように、MetalLBのバックエンドはデフォルトでfrr-k8sになります。

![](/images/metallb-frr-k8s-poc.png)
*検証環境のトポロジー*

MetalLB関連のPodは `metallb-system` namespaceで動きます。

```shell-session
$ oc -n metallb-system get pod
NAME                                                   READY   STATUS    RESTARTS   AGE
controller-79dcd8c4d8-7wb68                            2/2     Running   0          5d23h
metallb-operator-controller-manager-67d45c5969-gscf4   1/1     Running   0          5d23h
metallb-operator-webhook-server-5f849f45f7-z74g9       1/1     Running   0          5d23h
speaker-7gnk9                                          2/2     Running   0          5d23h
speaker-gn5sz                                          2/2     Running   0          5d23h
speaker-jhkwt                                          2/2     Running   0          5d23h
speaker-jlqz7                                          2/2     Running   0          5d23h
speaker-p2cxl                                          2/2     Running   0          5d23h
speaker-txn99                                          2/2     Running   0          5d23h
speaker-xpxkh                                          2/2     Running   0          5d23h
speaker-xvh82                                          2/2     Running   0          5d23h
```

speaker Podの中でfrrが動いていないことを確認します。

```shell-session
$ oc -n metallb-system get pod speaker-7gnk9 -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
speaker
kube-rbac-proxy
```

OpenShift v4.20 (もしくはv4.19.14以降) にMetalLB Operatorを入れると、自動的に `openshift-frr-k8s` namespaceができてそこにfrr-k8sのDaemonSetがデプロイされます。

```shell-session
$ oc -n openshift-frr-k8s get pod
NAME                                     READY   STATUS    RESTARTS       AGE
frr-k8s-2v6tv                            7/7     Running   25 (22d ago)   28d
frr-k8s-57svm                            7/7     Running   17 (22d ago)   28d
frr-k8s-7ztrw                            7/7     Running   14             28d
frr-k8s-9jzgv                            7/7     Running   26             28d
frr-k8s-bhshq                            7/7     Running   103            28d
frr-k8s-fv8k7                            7/7     Running   37             28d
frr-k8s-kbdgm                            7/7     Running   24 (22d ago)   28d
frr-k8s-vk2jh                            7/7     Running   396            28d
frr-k8s-webhook-server-7c886d9d4-khd4r   1/1     Running   214            28d
```

# 検証

## 検証1: MetalLB + frr-k8sでLoadBalancer Serviceを公開する (BGP経路はインポートしない)

まずは、WebサーバのPodとBGPで広告するLoadBalancer Serviceを作成します。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello
  name: hello-lb-l3
  annotations:
    metallb.io/address-pool: pool-l3
    metallb.io/loadBalancerIPs: 172.19.20.181
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    deployment: hello
  type: LoadBalancer
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: hello
  template:
    metadata:
      labels:
        deployment: hello
    spec:
      containers:
      - image: quay.io/manabu.ori/hello
        name: hello
      nodeSelector:
        node-role.kubernetes.io/worker-virt: ""
```

次に、いつものようにIPAddressPool、BGPPeer、BGPAdvertisementといったMetalLBカスタムリソースを設定します。ここでは、ラベル `node-role.kubernetes.io/worker-virt: ""` を設定しているノード (具体的にはwk3, wk4の2台) のみがBGPピアを張る設定にしています。

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: pool-l3
spec:
  addresses:
  - 172.19.20.181-172.19.20.189
  autoAssign: false
```

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: bgppeer-r2
  namespace: metallb-system
spec:
  myASN: 65801
  peerAddress: 172.18.20.1
  peerASN: 65102
  #ebgpMultiHop: true
  nodeSelectors:
  - matchLabels:
      node-role.kubernetes.io/worker-virt: ""
```

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgpadv1
  namespace: metallb-system
spec:
  ipAddressPools:
  - pool-l3
  peers:
  - bgppeer-r2
  nodeSelectors:
  - matchLabels:
      node-role.kubernetes.io/worker-virt: ""
```

ひととおり設定すると、LoadBalancer Serviceに対してExternal IP `172.19.20.181` が払い出されます。

```shell-session
$ oc get svc
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
hello-lb-l3   LoadBalancer   10.200.173.184   172.19.20.181   80:32240/TCP   6h56m
```

このとき、MetalLBのspeakerが各ノード用にカスタム `FRRConfiguration` を生成します。ラベル `node-role.kubernetes.io/worker-virt: ""` を設定したノードには、下記のようなFRRConfigurationが生成されます。

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  creationTimestamp: "2025-11-25T05:28:23Z"
  generation: 6
  name: metallb-wk3
  namespace: openshift-frr-k8s
  resourceVersion: "39144715"
  uid: 08112502-7df8-474f-9a8f-daeb0584407a
spec:
  bgp:
    routers:
    - asn: 65801
      neighbors:
      - address: 172.18.20.1
        asn: 65102
        disableMP: false
        dualStackAddressFamily: false
        passwordSecret: {}
        port: 179
        toAdvertise:
          allowed:
            mode: filtered
            prefixes:
            - 172.19.20.181/32
        toReceive:
          allowed:
            mode: filtered
      prefixes:
      - 172.19.20.181/32
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: wk3
  raw: {}
```

MetalLBのカスタムリソースBGPPeerやBGPAdvertisementで設定した内容が入っていることがわかります。ここで注目したいのは `spec.bgp.routers[0]` の `toAdvertise` と `toReceive` です。`toAdvertise` にはLoadBalancer ServiceのExternal IPがBGPで広告するprefixとして設定されています。`toRecieve` にはallwedなprefixがないので、外部からは何も経路を受け取らない設定になっていることがわかります。

frr-k8sはこのFRRConfigurationを見てfrrのコンフィグを生成します。frrのコンフィグは、frr-k8s Podのfrrコンテナに入ってvtyshして `show running config` してもいいですが、カスタムリソース `FRRNodeState` にも入っています。

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRNodeState
metadata:
  creationTimestamp: "2025-11-02T14:31:54Z"
  generation: 1
  name: wk3
  resourceVersion: "39144976"
  uid: c8804a52-b8d5-4661-8b55-fddb7ce8b2cd
spec: {}
status:
  lastConversionResult: success
  lastReloadResult: success
  runningConfig: |
    Building configuration...

    Current configuration:
    !
    frr version 8.5.3
    frr defaults traditional
    hostname wk3
    log file /etc/frr/frr.log informational
    log timestamp precision 3
    service integrated-vtysh-config
    !
    router bgp 65801
     no bgp ebgp-requires-policy
      no bgp default ipv4-unicast
     bgp graceful-restart preserve-fw-state
     no bgp network import-check
     neighbor 172.18.20.1 remote-as 65102
     !
     address-family ipv4 unicast
      network 172.19.20.181/32
      neighbor 172.18.20.1 activate
      neighbor 172.18.20.1 route-map 172.18.20.1-in in
      neighbor 172.18.20.1 route-map 172.18.20.1-out out
     exit-address-family
    exit
    !
    ip prefix-list 172.18.20.1-inpl-ipv4 seq 1 deny any   # ← ここ
    ip prefix-list 172.18.20.1-allowed-ipv4 seq 1 permit 172.19.20.181/32
    !
    ipv6 prefix-list 172.18.20.1-allowed-ipv6 seq 1 deny any
    ipv6 prefix-list 172.18.20.1-inpl-ipv4 seq 2 deny any
    !
    route-map 172.18.20.1-out permit 1
     match ip address prefix-list 172.18.20.1-allowed-ipv4
    exit
    !
    route-map 172.18.20.1-out permit 2
     match ipv6 address prefix-list 172.18.20.1-allowed-ipv6
    exit
    !
    route-map 172.18.20.1-in permit 3
     match ip address prefix-list 172.18.20.1-inpl-ipv4
    exit
    !
    route-map 172.18.20.1-in permit 4
     match ipv6 address prefix-list 172.18.20.1-inpl-ipv4
    exit
    !
    ip nht resolve-via-default
    !
    ipv6 nht resolve-via-default
    !
    end
```

`ip prefix-list 172.18.20.1-inpl-ipv4 seq 1 deny any` の設定が入っており、外部から受け取った経路を使わない設定になっていることがわかります。

## 検証2: 外部ルータから受け取ったBGP経路をインポートする

外部から受け取った経路をインポートするために、下のFRRConfigurationを適用します。

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  name: receive-all
  namespace: openshift-frr-k8s
spec:
  bgp:
    routers:
    - asn: 65801
      neighbors:
      - address: 172.18.20.1
        asn: 65102
        toReceive:
          allowed:
            mode: all
```

すると、frr-k8sがこのFRRConfigurationと、前述したMetalLBが生成するFRRConfigurationをいい感じにマージしたfrrのコンフィグを生成してくれます[^8]。カスタムリソースFRRNodeStateでrunning configを見ると、外部から受け取った経路のインポートを許可する設定になっていることがわかります。

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRNodeState
...
status:
    router bgp 65801
     no bgp ebgp-requires-policy
     no bgp default ipv4-unicast
     bgp graceful-restart preserve-fw-state
     no bgp network import-check
     neighbor 172.18.20.1 remote-as 65102
     !
     address-family ipv4 unicast
      network 172.19.20.181/32
      neighbor 172.18.20.1 activate
      neighbor 172.18.20.1 route-map 172.18.20.1-in in
      neighbor 172.18.20.1 route-map 172.18.20.1-out out
     exit-address-family
    exit
    !
<snip>
    ip prefix-list 172.18.20.1-inpl-ipv4 seq 1 permit any   # ←ここ
<snip>
    route-map 172.18.20.1-in permit 3
     match ip address prefix-list 172.18.20.1-inpl-ipv4
    exit
<snip>
```

ノード `wk3` のfrrで `show ip bgp` すると下記のようになります。

```shell-session
$ oc -n openshift-frr-k8s exec frr-k8s-vk2jh -c frr -- vtysh -c 'show ip bgp'
BGP table version is 4, local router ID is 172.18.20.113, vrf id 0
Default local pref 100, local AS 65801
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *> 0.0.0.0/0        172.18.20.1                            0 65102 65101 i
 *> 172.18.20.0/24   172.18.20.1              0             0 65102 i
 *> 172.18.30.0/24   172.18.20.1                            0 65102 65101 65103 i
 *> 172.19.20.181/32 0.0.0.0                  0         32768 i

Displayed  4 routes and 4 total paths
```

ノード `wk3` 上で `ip route show` すると、受け取った経路がカーネルのRIBに入っていることがわかります。

```shell-session
[core@wk3 ~]$ ip route show proto bgp
172.18.30.0/24 nhid 12186 via 172.18.20.1 dev br-ex metric 20
```

# まとめ

MetalLB + frr-k8sを使っているときに、外部のルータから受け取ったBGP経路をインポートする方法のご紹介でした。

[^1]: https://github.com/metallb/metallb/issues/251
[^2]: https://www.redhat.com/ja/blog/learning-kubernetes-nodes-networking-routes-via-bgp
[^3]: https://github.com/metallb/metallb/commit/f389862268d13604506c0d571c889f83acfc5ac8
[^4]: https://www.redhat.com/ja/blog/frr-k8s-bgp-backend-metallb
[^5]: https://metallb.io/installation/
[^6]: https://metallb.io/concepts/bgp/#frr-k8s-mode
[^7]: 図にはShowNetアイコンを使用させていただきました https://github.com/interop-tokyo-shownet/shownet-icons
[^8]: FRRConfigurationをマージするときの方針はfrr-k8sのREADMEに概要が載っています https://github.com/metallb/frr-k8s?tab=readme-ov-file#how-multiple-configurations-are-merged-together

