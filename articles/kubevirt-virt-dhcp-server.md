---
title: "OpenShift Virtualizationで誰がDHCPのアドレスを配っているか"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [kubevirt, openshift, kubernetes]
published: false
---
OpenShift Virtualization Advent Calendar 2日目の記事です。

仮想化基盤やクラウド環境で使用する仮想マシンイメージでは、汎用性を高めるために、NICのIPアドレスはDHCPで取得するよう設定することが多いと思います。つまり、仮想化インフラのネットワーク側でDHCPでIPアドレスを配布する仕組みを提供する必要があります。KubeVirtはVM(libvirt+qemu)をPodに閉じ込めてKubernetes上でVMが動く仕組みですが、さてKubernetesのPodネットワークでDHCPを配るのはどうしているのでしょうか...という疑問に答える記事です。

Kubernetesにおいて、ホストとPodを接続する役割を担うのはCNIプラグインです。KubeVirtにおいては、virt-launcherと呼ばれるPodの中にVMがいるので、そこを接続する仕組みが必要です。このPodとVMを接続する部分をKubeVirtではNetwork Bindingと呼んでいます。

Network Bindingの仕組みは、KubeVirtのin-treeに含まれるcore binding modeは以下の3つです。

- bridge
- sriov
- masquerade

in-treeではなくプラグインとして提供されるものもあります。

- passt
- macvtap
- slirp

OpenShift Virtualizationの観点では、core bindingに加えてpasstがTechPreviewとして提供されています。

本記事では、通常のPodネットワークに接続するときにデフォルトで使われるmasquerade、UDN接続時に使われるbridge、の2つのnetwork binding使用時のDHCPについて説明します。

# network bindingがmqsqueradeのとき

仮想マシンのインターフェースは、virt-launcher Pod内でNATしてPodネットワークに接続します。通常のPodネットワークに接続するときのデフォルトのnetwork bindingです。virt-launcher Podの中にLinux bridge `k6t-eth0` があり、VMが接続するTAPデバイスがそこに接続します。またそのブリッジからPodのeth0にひねるnftablesのルールがvirt-launcher Podのnetwork namespaceに設定されます。NATの内側は `10.0.2.0/24` のアドレスになっており、VMには通常 `10.0.2.2` のアドレスが割り当てられます。

ネットワーク構成としては下記のようになります[^1]。

[^1]: https://kubevirt.io/2018/KubeVirt-Network-Deep-Dive.html#kubevirt-networking

![](https://kubevirt.io/assets/images/diagram.png)
*masquerade使用時のvirt-launcher Pod内のネットワーク構成*

https://speakerdeck.com/orimanabu/kubevirt-networking-onic-2024?slide=12

このnetwork bindingの場合、仮想マシンに対するDHCPサーバの役割はvirt-launcher Podの中で稼働する `virt-launcher` プロセスのスレッド(goroutine)が担います。仮想マシンを起動したときのvirt-launcher Podのログを見ると、下記のようなメッセージが出力されており、DHCPサーバが動いていることがわかります。このDHCPサーバはとても簡易的な実装で、該当VMは1台のアドレスを配ることしかしません。

```json
{"component":"virt-launcher","level":"info","msg":"The imported dhcpConfig: DHCPConfig: { Name: eth0, IPv4: 10.0.2.2/24, IPv6: <nil>, MAC: , AdvertisingIPAddr: 10.0.2.1, MTU: 1400,
 Gateway: 10.0.2.1, IPAMDisabled: false, Routes: <nil>}","pos":"podnic.go:104","timestamp":"2025-12-01T21:32:37.036345Z"}
{"component":"virt-launcher","level":"info","msg":"StartDHCP network Nic: DHCPConfig: { Name: eth0, IPv4: 10.0.2.2/24, IPv6: <nil>, MAC: , AdvertisingIPAddr: 10.0.2.1, MTU: 1400, G
ateway: 10.0.2.1, IPAMDisabled: false, Routes: <nil>}","pos":"common.go:162","timestamp":"2025-12-01T21:32:37.036402Z"}
{"component":"virt-launcher","level":"info","msg":"Found nameservers in /etc/resolv.conf: \n\ufffd\u0000\n","pos":"resolveconf.go:168","timestamp":"2025-12-01T21:32:37.036518Z"}
{"component":"virt-launcher","level":"info","msg":"Found search domains in /etc/resolv.conf: proj1.svc.cluster.local svc.cluster.local cluster.local ovnk.setns.net","pos":"resolvec
onf.go:169","timestamp":"2025-12-01T21:32:37.036540Z"}
{"component":"virt-launcher","level":"info","msg":"Starting SingleClientDHCPServer","pos":"server.go:65","timestamp":"2025-12-01T21:32:37.036605Z"}
```

# newtork bindingがbridgeのとき

仮想マシンのインターフェースはvirt-laucher Pod内のLinux bridgeを介してPodネットワークにL2接続します。

![](/images/kubevirt-dhcp-1.png)
*bridge使用時のvirt-launcher Pod内のネットワーク構成*

仮想マシンが起動すると、ゲストOS起動時にDHCPリクエストをブロードキャストします。network bindingがbridgeモードのときは、PodネットワークまでL2でつながっているため、Podネットワーク上にDHCPリプライを返すDHCPサーバが必要です。

CNIプラグインがOVN-Kubernetesの場合、User Defined Network (以下UDN、うどん) という独自にテナントネットワークを作成してPodやVMを接続する仕組みがあります。
UDNを作った場合は、OVNが提供するオーバーレイネットワーク用DHCPサービスの機能を利用して、仮想的にDHCPサービスを提供します。

仮想マシンが起動してゲストOSがDHCPリクエストを投げると、OVNのロジカルフローでプログラムされたDHCPルールにマッチして、ovn-controllerがDHCPリプライを返す、という動きになります。

例えばLayer2トポロジーとしてCluster UDN `cudn2` を作った場合に自動的に作成されるDHCPルールは下記のようになります。

``` shell-session
$ oc -n openshift-ovn-kubernetes exec -c ovn-controller ${ovnk_pod} -- ovn-sbctl dump-flows cluster_udn_cudn2_ovn_layer2_switch | grep -E 'udp.*6[78]'
  table=23(ls_in_dhcp_options ), priority=100  , match=(inport == "proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv" && eth.src == 0a:58:ac:16:00:06 && (ip4.src == {172.22.0.6, 0.0.0.0} && ip4.dst == {169.254.1.1, 255.255.255.255}) && udp.src == 68 && udp.dst == 67), action=(reg0[3] = put_dhcp_opts(offerip = 172.22.0.6, dns_server = 10.200.0.10, hostname = "fedora-1", lease_time = 3500, mtu = 1400, netmask = 255.255.0.0, router = 172.22.0.1, server_id = 169.254.1.1); next;)
  table=24(ls_in_dhcp_response), priority=100  , match=(inport == "proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv" && eth.src == 0a:58:ac:16:00:06 && ip4 && udp.src == 68 && udp.dst == 67 && reg0[3]), action=(eth.dst = eth.src; eth.src = 0a:58:a9:fe:01:01; ip4.src = 169.254.1.1; udp.src = 67; udp.dst = 68; outport = inport; flags.loopback = 1; output;)
  table=6 (ls_out_acl_eval    ), priority=34000, match=(outport == "proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv" && eth.src == 0a:58:a9:fe:01:01 && ip4.src == 169.254.1.1 && udp && udp.src == 67 && udp.dst == 68), action=(reg8[16] = 1; next;)
```

ovn-traceを実行すると下記のようになります。

``` shell-session
$ oc -n openshift-ovn-kubernetes exec -c ovn-controller ${ovnk_pod} -- ovn-trace --summary --ct new 'inport == "'${ovn_port}'" && eth.src == '${vm_mac}' && ip4.src == 0.0.0.0 && ip4.dst == 255.255.255.255 && udp.src == 68 && udp.dst == 67'| grep -Ev 'reg|next;'
ingress(dp="cluster_udn_cudn2_ovn_layer2_switch", inport="proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv") {
    ct_lb_mark;
    ct_lb_mark {
        /* We assume that this packet is DHCPDISCOVER or DHCPREQUEST. */;
        eth.dst = eth.src;
        eth.src = 0a:58:a9:fe:01:01;
        ip4.src = 169.254.1.1;
        udp.src = 67;
        udp.dst = 68;
        outport = inport;
        flags.loopback = 1;
        output;
        egress(dp="cluster_udn_cudn2_ovn_layer2_switch", inport="proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv", outport="proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv") {
            ct_lb_mark;
            ct_lb_mark /* default (use --ct to customize) */ {
                output;
                /* output to "proj2.cudn2_proj2_virt-launcher-fedora-1-hs6zv", type "" */;
            };
        };
    };
};
```

tcpdumpするると、下記のようなDHCPのやり取りが見えます。

```shell-session
22:28:25.860212 0a:58:ac:16:00:06 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 334: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 0a:58:ac:16:00:06, length 292
22:28:25.861925 0a:58:a9:fe:01:01 > 0a:58:ac:16:00:06, ethertype IPv4 (0x0800), length 338: 169.254.1.1.67 > 172.22.0.6.68: BOOTP/DHCP, Reply, length 296
```

`169.254.1.1` というアドレスがDHCPサーバのように見えます。これはOVN-Kubernetesがデフォルトゲートウェイ兼DHCPサーバとして内部で使用しているアドレスです。

https://github.com/ovn-kubernetes/ovn-kubernetes/blob/e2532461ebfb36deab486b2519e5bbdf1b231df9/go-controller/pkg/kubevirt/dhcp.go#L148-L156
https://github.com/ovn-kubernetes/ovn-kubernetes/blob/e2532461ebfb36deab486b2519e5bbdf1b231df9/go-controller/pkg/kubevirt/switch.go#L10-L12

# まとめ

OVN-KubernetesのUDNを使った場合、network bindingがbridgeのときに誰がDHCPアドレスを配っているかを調べたところ、OVNのDHCP機能を使っていることがわかりました。他のCNIプラグインを使う場合は、何らかの形でDHCPサーバを用意する必要があります。
