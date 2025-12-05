---
title: "OVN-Kubernetesで今後実装されそうな機能(2025年度版)"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Kubernetes, OpenShift, OVN-Kubernetes]
published: true
---
# はじめに

OpenShift Advent Calendar 2025の5日目です。OVN-Kubernetesには、[OKEP](https://github.com/ovn-kubernetes/ovn-kubernetes/tree/master/docs/okeps) (OVN-Kubernetes Enhancement Proposal) というドキュメントがあります。Kubernetesにおける[KEP](https://github.com/kubernetes/enhancements/tree/master/keps) (Kubernetes Enhancement Proposal) と同じように、新機能を開発する際はまずデザインドキュメント的なものをOKEPという形で書き、その上で実際のコードを書きます。本記事では、いくつかの進行中のOKEPを眺めて、今後実装されるかもしれない機能をチラ見してみます。

# トラブルシュート用MCPサーバー

OKEPはこれです: OKEP-5494: Model Context Protocol for Troubleshooting OVN-Kubernetes

https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/docs/okeps/okep-5494-ovn-kubernetes-mcp-server.md

CNIプラグインOVN-Kubernetesのトラブルシュートは、Kubernetes、OVN-Kubernetes、OVN、Open vSwitch、カーネル(特にNetfilter)にまたがる総合格闘技で、とても複雑で難しいことが多いです。関連ツールも多岐にわたり、`kubectl`、`ovn-nbctl`、`ovs-ofctl`、`ip`、`nft`、`tcpdump` 等のツールを使いこなして情報を収集し、分析する必要があります。この課題に対応するためのOVN-KubernetesのMCPサーバを作ろう、というのがこのOKEPです。具体的には、ユーザーからの問い合わせをMCPサーバを経由してLLMに送り、LLMはMCPサーバが公開したツールから適切なものを選択して実行を指示し、その出力を分析して根本原因をユーザーに提供することを目標にしています。環境へのアクセスはread onlyであることを保証し、またmust-gatherやsosreport等の資料 (Red Hat製品の情報をまるっと取得するツールの出力) を使ったトラブルシュートのサポートもスコープに入っています。

MCPサーバーのまだ実装が始まったばかりですが、こちらで開発が進んでいます。

https://github.com/ovn-kubernetes/ovn-kubernetes-mcp

# no-overlay

OVN-KubernetesのBGP対応により、デフォルトPodネットワークやUDNがBGPで広告できるようになっています。この場合、North-South通信 (クラスターの外との通信) はNATせず直接外に出ていきますが、East-West通信 (Pod間通信) は従来どおりGeneveでencapしてノードをまたぎます。このOKEPが実装されると、East-West通信でGeneveトンネルを使わず、外部のルータを通すことでパフォーマンスの向上が図れる、とのことです。

https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/docs/okeps/okep-5259-no-overlay.md

余談ですが、本記事執筆時は下のOKEP PRのURLを紹介しようと思っていたのですが、ちょうど昨日PRがマージされ、正式なOKEPとしてリリースされたのでした。

https://github.com/ovn-kubernetes/ovn-kubernetes/pull/5289

APIとしては、カスタムリソースClusterUserDefinedNetworkを拡張する(`noOverlayOptions`設定が追加される)形になります。こちらもちょうど昨日[PR](https://github.com/ovn-kubernetes/ovn-kubernetes/pull/5644)が出ています。

# ClusterNetworkConnect

こちらは10月末にマージされたOKEPです。

https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/docs/okeps/okep-5224-connecting-udns/okep-5224-connecting-udns.md

OVN-Kubernetesを使ったKubernetes/OpenShift環境では、User Defined Network (UDN) や Cluster User Defined Network (CUDN) により、テナントごとに独自のPodネットワークを作成することができるようになりました。ClusterNetworkConnectのOKEPは、複数のUDN/CUDNを接続したり切断したりできるようにする機能です。OVNのオーバーレイネットワーク上のネットワークトポロジーにおいて、カスタムリソースClusterNetworkConnectごとに論理ルーターconnect-routerを作成し、UCN/CUDN間を接続する、という仕組みになっています。

# OVS-DPDK関連

去年と今年のOVS+OVN Conferenceで講演があったように、OVS-DPDKをOpenShift上で動かす取り組みが進んでいます。OVS-DPDKにコンテナとVMの両方を接続できるようにするために、従来のvhost-user経由ではなくvDPA in Userspace (VDUSE)という仕組みを導入する方向で実装が進んでいます。

- [OpenShift Networking Transformed: Fully Embracing DPDK Datapaths in OVN-K8s!](https://www.openvswitch.org/support/ovscon2024/slides/OpenShift%20Networking%20Transformed.pdf), OVS+OVN Conf 2024
- [Beam Me Through the Datapath: VDUSE for OpenShift Virtualization](https://www.openvswitch.org/support/ovscon2025/slides/OVSOVNCON25%20-%20Beam%20Me%20Through%20the%20Datapath_%20VDUSE%20for%20OpenShift%20Virtualization.pdf), OVS+OVN Conf 2025


この流れで、OVN-Kubernetes側にもvDPA対応の実装が入り始めています。

vhost-vDPA support #3958
https://github.com/ovn-kubernetes/ovn-kubernetes/pull/3958

Add support to virtio-vDPA devices #2664
https://github.com/ovn-kubernetes/ovn-kubernetes/pull/2664

# EVPN

BGPベースのネットワークを作ると、そこにL2で抜きたくなるのが人情なのか、EVPN対応のOKEPが提案されています(まだマージされていません)。

PR: https://github.com/ovn-kubernetes/ovn-kubernetes/pull/5089

内容: https://github.com/ovn-kubernetes/ovn-kubernetes/blob/3222346ed79d853634068288ce3685767ee35210/docs/okeps/okep-5088-evpn.md

L2トポロジーのUDNを外部までL2で延伸するのが主なユースケースです。OVNはノード間通信をGeneveでencapしていましたが、EVPNを有効化すると、GeneveではなくVXLANでカプセル化します。つい昨日、ちょうどカスタムリソースEVPN, VTEPの[Draft PR](https://github.com/ovn-kubernetes/ovn-kubernetes/pull/5779)がsubmitされていました

本OKEPの最後に「OpenPERouterを使う方法もあったが、いろいろ考えた結果OpenPERouterは使わずOVN-Kubernetes独自にEVPNを実装することにした」と書いてあります (ﾅ､ﾅﾝﾀﾞｯﾃｰ)。

OpenPERouterはこれはこれで興味深いプロジェクトなのですが、今回は参考リンクを載せておくにとどめて、気が向いたら改めて記事を書きたいと思います。

- https://openperouter.github.io/ (プロジェクトのホームページ)

- https://kubevirt.io/2025/Stretched-layer2-network-between-clusters.html (KubeVirtでの使用例)

# まとめ

いくつかのOKEP (未マージのもの含む) の概要を紹介しました。OVN-Kubernetesは活発に機能追加が継続しているCNIプラグインのひとつです。去年[CNCF Sandboxプロジェクト](https://www.cncf.io/projects/ovn-kubernetes/)になり、最近はNVIDIAはじめRed Hat以外の方々も積極的に開発に参加しています。ご興味のある方は、ぜひ[PR](https://github.com/ovn-kubernetes/ovn-kubernetes/pulls)やSlackチャネル(workspace: https://cloud-native.slack.com/, channel: #ovn-kubernetes)を眺めてみてください。
