---
title: "PodmanのコンテナからmacOSのApple Silicon GPUを使ってAIワークロードを高速処理できるようになりました"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["podman", "macOS", "libkrun", "GPU"]
published: true
---
# はじめに

macOS上のPodmanで、コンテナからApple Silicon MacのGPUにアクセスすることができるようになりました。macOS上でPodmanを実行する際は、実際はPodman Machineと呼ばれるLinux仮想マシンが起動し、その中で動くPodmanに対してREST APIで接続することで、Linuxコンテナを実行します。「コンテナからMacのGPUにアクセスする」ということは、Podman Machine (つまりmacOS上の仮想マシン) からGPUにアクセスすることを意味します。

macOS上でコンテナを動かすためのツールとしては、DockerやLima + containerd + nerdctlが挙げられますが、おそらく2024年8月現在、コンテナからApple Silicon GPUを使えるのはPodmanだけです(私が調べた限り...うそだったらごめんなさい)。

(2024-10-30追記) Docker v4.35.0から、Docker VMMがBetaリリースされました。Docker VMMはlibkrunを使っているようです

https://x.com/orimanabu/status/1851262575436247384

試してみたところ、本記事と同じ方法でコンテナからGPUを使うことができました。

```
ori@macbookair ~ % docker version -f json | jq -r .Server.Platform.Name
Docker Desktop 4.35.0 (172550)
ori@macbookair ~ % docker run --rm -ti --device /dev/dri -v ~/Downloads:/models:Z quay.io/slopezpa/fedora-vgpu-llama main --temp 0 -m models/mistral-7b-instruct-v0.2.Q4_0.gguf -b 512 -ngl 99 -p "Tell me a story"
Log start
main: build = 2238 (56d03d92)
main: built with cc (GCC) 13.2.1 20231205 (Red Hat 13.2.1-6) for aarch64-redhat-linux
main: seed  = 1730218047
ggml_vulkan: Found 1 Vulkan devices:
Vulkan0: Virtio-GPU Venus (Apple M2) | uma: 1 | fp16: 1 | warp size: 32
...
llama_print_timings:        load time =    8999.20 ms
llama_print_timings:      sample time =      39.70 ms /   534 runs   (    0.07 ms per token, 13450.54 tokens per second)
llama_print_timings: prompt eval time =    3929.67 ms /     5 tokens (  785.93 ms per token,     1.27 tokens per second)
llama_print_timings:        eval time =   68185.57 ms /   533 runs   (  127.93 ms per token,     7.82 tokens per second)
llama_print_timings:       total time =   72243.51 ms /   538 tokens
Log end
```

(追記ここまで)

本記事は、2部構成になっています。

- 前半: Podman + libkrunでコンテナからGPUを利用する方法
- 後半: Podman + libkrunによってコンテナからGPUを利用できるようになる仕組み、もしくはなぜDockerやLimaではコンテナ(Linux仮想マシン)からGPUが使えないか

# Podman + libkrunでコンテナからGPUを利用する方法

2部構成の前半、とりあえず実際に動かしてみます。

## 準備

セットアップ方法は大きく3種類あります。
1. brewを使ってpodman, krunkitをインストールする
2. pkg形式のインストーラを使用する
3. Podman Desktopからセットアップする

### brewを使ったインストール

["Podman and Libkrun"](https://blog.podman.io/2024/07/podman-and-libkrun/) というブロク記事にしたがって、podmanとkrunkitをbrew installします。

https://blog.podman.io/2024/07/podman-and-libkrun/

```sh
% brew install podman
```

```sh
% brew tap slp/krunkit
% brew install krunkit
```

さらに、`~/.config/containers/containers.conf` に以下を記述します (このファイルがない場合は作成してください)。

```ini
[machine]
provider="libkrun"
```

最後にPodman Machineを作成＆起動します。

```sh
% podman machine init --now
```

(`podman machine init --now` は、`podman machine init` して `podman machine start` するのを一度にやるコマンドです)

### pkg形式のインストーラを使ったインストール

v5.2以降であれば、[公式のリリースページ](https://github.com/containers/podman/releases)にあるpkg形式のインストーラ(本記事執筆時点の最新版はv5.2.2の[podman-installer-macos-arm64.pkg](https://github.com/containers/podman/releases/download/v5.2.2/podman-installer-macos-arm64.pkg))をダウンロードして実行します。
Podmanだけでなくkrunkitも自動的にインストールされます。

インストールが終わったら、brewで入れたときと同じように `~/.config/containers/containers.conf` を作成し、`podman machine init --now` を実行します。

### Podman Desktopを使ったセットアップ

PodmanのGUIアプリであるPodman Desktopも、 v1.12からlibkrunを使ったPodman Machineに対応しています。

https://podman-desktop.io/blog/podman-desktop-release-1.12

(どうでもいいですが、このグラボを持ったセルキーかわいいですね)

Podman Desktopのインストール方法は[公式ドキュメント](https://podman-desktop.io/docs/installation/macos-install)をご参照ください (dmgをダウンロードする、もしくはbrewを使います)。

Podman Desktop v1.12をご使用の場合は、Podman Machineを作成するときに Provider Type として `GPU enabled (LibKrun)` を選択すると、libkrunを使ったPodman Machineが起動します。

![](/images/podman-desktop-machine-create-libkrun.png)

## 確認

上記のいずれかの方法でセットアップすると、libkrun providerを使ったPodman Machineが起動します。このPodman Machineに入ると、`/dev/dri` 以下にGPU関連のデバイスファイルが生えていることがわかります。

```sh
% podman machine ssh
Connecting to vm podman-machine-default. To close connection, use `~.` or `exit`
Fedora CoreOS 40.20240808.2.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

core@localhost:~$ find /dev/dri -type c -or -type l
/dev/dri/renderD128
/dev/dri/card0
/dev/dri/by-path/platform-a007000.virtio_mmio-render
/dev/dri/by-path/platform-a007000.virtio_mmio-card
core@localhost:~$ exit
logout
```

念のため、(GPUを使わない普通の)コンテナを実行できることを確認しておきます。

```sh
% podman run --rm hello-world
Resolved "hello-world" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull quay.io/podman/hello:latest...
Getting image source signatures
Copying blob sha256:1ff9adeff4443b503b304e7aa4c37bb90762947125f4a522b370162a7492ff47
Copying config sha256:83fc7ce1224f5ed3885f6aaec0bb001c0bbb2a308e3250d7408804a720c72a32
Writing manifest to image destination
!... Hello Podman World ...!

         .--"--.
       / -     - \
      / (O)   (O) \
   ~~~| -=(,Y,)=- |
    .---. /`  \   |~~
 ~/  o  o \~~~~.----. ~~
  | =(X)= |~  / (O (O) \
   ~~~~~~~  ~| =(Y_)=-  |
  ~~~~    ~~~|   U      |~~

Project:   https://github.com/containers/podman
Website:   https://podman.io
Desktop:   https://podman-desktop.io
Documents: https://docs.podman.io
YouTube:   https://youtube.com/@Podman
X/Twitter: @Podman_io
Mastodon:  @Podman_io@fosstodon.org
```

## AI/MLワークロードの実行

llama.cppを使ったAI/MLワークロードをコンテナで実行してみます (現時点の実装では、GPGPUとしては利用できますが、3Dグラフィックスのアクセラレーションはできません)。

事前にモデルを `~/Downloads` にダウンロードしておきます (下の例では `Llama-3-ELYZA-JP-8B-q4_k_m.gguf` を使っています)。
`/dev/dri` をホストデバイスとしてコンテナに追加し(ここで言う「ホスト」はPodman Machineのことです)、さらに `~/Downloads` をコンテナ内にbind mountしてPodmanを実行します。

```sh
% podman run --rm -ti --device /dev/dri -v ~/Downloads:/models:Z quay.io/slopezpa/fedora-vgpu-llama bash
```

コンテナ内で `vulkaninfo` を実行すると、`Virtio-GPU Venus (Apple M2)` として準仮想化GPUデバイスが見えていることがわかります。

```
[root@5528136b7365 /]# vulkaninfo --summary

<略>

Devices:
========
GPU0:
	apiVersion         = 1.2.0
	driverVersion      = 23.3.5
	vendorID           = 0x106b
	deviceID           = 0xe0603f0
	deviceType         = PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU
	deviceName         = Virtio-GPU Venus (Apple M2)
	driverID           = DRIVER_ID_MESA_VENUS
	driverName         = venus
	driverInfo         = Mesa 23.3.5
	conformanceVersion = 1.3.0.0
	deviceUUID         = 901f9c56-e0b4-ba61-62fb-f05d68e99066
	driverUUID         = d2076857-3352-aaf9-a5ba-778bcb213a61

<略>
```

llama.cppで推論をしてみます。

```
[root@5528136b7365 /]# main --temp 0 -m models/Llama-3-ELYZA-JP-8B-q4_k_m.gguf -b 512 -ngl 99 -p "Podmanのlibkrun providerについて教えて下さい"
Log start
main: build = 2238 (56d03d92)
main: built with cc (GCC) 13.2.1 20231205 (Red Hat 13.2.1-6) for aarch64-redhat-linux
main: seed  = 1725466249
ggml_vulkan: Found 1 Vulkan devices:
Vulkan0: Virtio-GPU Venus (Apple M2) | uma: 1 | fp16: 1 | warp size: 32
llama_model_loader: loaded meta data with 22 key-value pairs and 291 tensors from models/Llama-3-ELYZA-JP-8B-q4_k_m.gguf (version GGUF V3 (latest))
llama_model_loader: Dumping metadata keys/values. Note: KV overrides do not apply in this output.
llama_model_loader: - kv   0:                       general.architecture str              = llama
llama_model_loader: - kv   1:                               general.name str              = Llama-3-8B-optimal-merged-stage2

<略>

system_info: n_threads = 4 / 4 | AVX = 0 | AVX_VNNI = 0 | AVX2 = 0 | AVX512 = 0 | AVX512_VBMI = 0 | AVX512_VNNI = 0 | FMA = 0 | NEON = 1 | ARM_FMA = 1 | F16C = 0 | FP16_VA = 0 | WASM_SIMD = 0 | BLAS = 1 | SSE3 = 0 | SSSE3 = 0 | VSX = 0 | MATMUL_INT8 = 0 |
sampling:
	repeat_last_n = 64, repeat_penalty = 1.100, frequency_penalty = 0.000, presence_penalty = 0.000
	top_k = 40, tfs_z = 1.000, top_p = 0.950, min_p = 0.050, typical_p = 1.000, temp = 0.000
	mirostat = 0, mirostat_lr = 0.100, mirostat_ent = 5.000
sampling order:
CFG -> Penalties -> top_k -> tfs_z -> typical_p -> top_p -> min_p -> temperature
generate: n_ctx = 512, n_batch = 512, n_predict = -1, n_keep = 0


Podmanのlibkrun providerについて教えて下さい。

libkrunはKDEのRun Commandを提供するライブラリです。libkrun providerは、libkrunが提供するRun Commandを実行するためのプロバイダーです。

libkrun providerは、libkrunが提供するRun Commandを実行するために必要な情報を提供します。例えば、実行するコマンドのパスや環境変数などです。

<略>

llama_print_timings:        load time =   16897.88 ms
llama_print_timings:      sample time =     475.69 ms /   307 runs   (    1.55 ms per token,   645.38 tokens per second)
llama_print_timings: prompt eval time =    4442.21 ms /    12 tokens (  370.18 ms per token,     2.70 tokens per second)
llama_print_timings:        eval time =   26072.46 ms /   306 runs   (   85.20 ms per token,    11.74 tokens per second)
llama_print_timings:       total time =   31345.05 ms /   318 tokens
Log end
```

無事実行することができました (回答はデタラメですが)。macOS上でアクティビティモニタを見ると、実際にGPUを使っていることがわかります。

![](/images/podman-libkrun-gpu-1.png)
![](/images/podman-libkrun-gpu-2.png)

# コンテナからGPUを使う仕組み

2部構成の後半です。ここからは、libkrunを使うとコンテナからGPUを使えるようになる技術的な背景などを書きます。

## なぜlibkrunか

冒頭で書いたとおり、PodmanをmacOSで使うときは、Linux仮想マシンを起動し、その仮想マシンに対してREST API接続することでコンテナを実行します。Podmanはv0.8.2からmacOSに対応していますが、Podman Machineの実装としては、

- v0.8.2〜v4.9: Qemu + Hypervisor.framework (`-accel hvf` つきで起動します[^1])
- v5.0以降: Virtualization.framework

[^1]: https://github.com/containers/podman/blob/v4.9.5/pkg/machine/qemu/options_darwin_arm64.go#L18

をVMMとして使います。

Podman MachineからGPUを使う方法を検討したところ、いずれの方法でも短期的には解決できそうにない課題があったため、第3の選択肢として、libkrunをVMMとして使うことになりました。

まず最初にQemu、Virtualization.frameworkそれぞれの課題を整理...する前に、前提知識としてmacOSの仮想化機構について簡単にまとめます。

## macOSの仮想化機構

macOSは、仮想化に関して[Hypervisor.framework](https://developer.apple.com/documentation/hypervisor)、[Virtualization.framework](https://developer.apple.com/documentation/virtualization)という2つのフレームを持っています。Hypervisor.frameworkは主にCPUやメモリの仮想化などの低レイヤ/カーネル側を担当するフレームワークで、LinuxにおけるKVMに相当します。Virtualization.frameworkは主に仮想デバイス等を提供する高レイヤ/ユーザーランド側のフレームワークで、LinuxにおけるQemuに相当します。便宜上、本記事では前者を「ハイパーバイザー」、後者を仮想マシンモニター(VMM)と呼びます。

||macOS|Linux
|---|---|---|
|ハイパーバイザー(CPU/メモリの仮想化等)|Hypervisor.framework|KVM|
|VMM(仮想デバイスの管理等)|Virtualization.framework|Qemu|

![](/images/Hypervisor_framework_and_KVM.png)
*macOSのHypervisor.frameworkとLinux KVM[^2]*

[^2]: 図はKVM Forum 2022のセッション「[Hypervisor.Framework - Virtualization on macOS](https://www.youtube.com/watch?v=adTjIMXjBLc)」より引用 (リンク先は動画です、スライドは公開されていないようでした)

UTM, LimaといったmacOS上でよく使われる仮想化管理ツールの多くは、Virtualization.frameworkを使っています (設定でQemuと選べますがだいたいVirtualization.frameworkを使いますよね...きっと)。対して、xhyveやQemuはHypervisor.frameworkを使います。

||Hypervisor.framework|Virtualization.framework|
|---|---|---|
|仮想化管理ツール|xhyve, Qemu, libkrun, ...|vfkit, UTM, Lima, ...|

![](/images/Hypervisor_framework_and_Qemu.png)
*macOS上のQemuとHypervisor.framework[^3]*
![](/images/Virtualization_framework.png)
*macOS上のVirtualization.framework[^4]*

[^3]:図はKVM Forum 2023のセッション「[macOS in QEMU (ARM edition)](https://kvm-forum.qemu.org/2023/macOS_in_QEMU_on_ARM_FhJY65D.pdf)」のp.3より引用
[^4]: 図はKVM Forum 2023のセッション「[macOS in QEMU (ARM edition)](https://kvm-forum.qemu.org/2023/macOS_in_QEMU_on_ARM_FhJY65D.pdf)」のp.4より引用

## Linuxゲストに対するGPUサポート

ざっと調べた限り、様々な仮想化ツールで、macOS上のLinuxゲストからGPUを使うことについて議論されています。
- machyve/xhyve: [Device Passthrough ( Most notably, GPU ) #108](https://github.com/machyve/xhyve/issues/108)
- moby/hyperkit: [PCI-passthrough/GPU support #159](https://github.com/moby/hyperkit/issues/159)
- utmapp/UTM: [Virgl not functionning under Apple's virtualization.framework #5482](https://github.com/utmapp/UTM/discussions/5482)
- ブログ記事:「[Apple Silicon GPUs, Docker and Ollama: Pick two.](https://chariotsolutions.com/blog/post/apple-silicon-gpus-docker-and-ollama-pick-two/)」

Podman MachineからGPUを使うにあたり、VMMとしてQemuを使った場合とVirtualization.frameworkを使う場合のそれぞれについて、どういう課題があるかをまとめました。


## Qemuの課題

Apple Siliconが出た当初から使われている代表的なVMMとして、Qemuがあります。しかし、Qemuにはいくつかの課題があります。

- TSO (Total Store Ordering) / Rosettaサポートがない
- virtiofsがないので、virtio-9pを使うしかない (virtio-9pは遅いしSELinuxサポートに課題)

macOS用Podmanは、上で書いたように最初はQemuを使っていましたが、v5.0からVirtualization.frameworkを使った仮想化に切り替えました。理由は、Qemuの不具合でしばしばmacOS用Podmanが動かなくなり、[その対応に苦慮していた](https://github.com/containers/podman/pull/21145#issuecomment-1913717227)ためです。

QemuのLinuxゲストからmacのGPUを使う実装はあるようです。

https://gist.github.com/akihikodaki/87df4149e7ca87f18dc56807ec5a1bc5

https://github.com/knazarov/homebrew-qemu-virgl

しかし、Podman MachineでGPUを使う方法を検討していたのは2024年初頭で、当時既にVirtualization.frameworkベースのPodman Machineに移行することになっていたこともあり(2024年3月末にVirtualization.frameworkに移行したv5.0が出ました)、PodmanのGPU対応をQemuで頑張るのは避けたかったようです。

## Virtualization.frameworkの課題

v5.0以降でPodman MachineはVirtualization.frameworkベースに移行し、Qemuに起因する課題の多くは解決しました。

- VMMとして安定している
- virtiofsが使える
- Rosettaが使える

一方で、いくつか新たな課題が見えてきました。

- GPUを使えない
- Virtualization.frameworkが提供するvirtiofsの課題がある

実は、Virtualization.frameworkは準仮想化GPUデバイスを提供していますが([ParavirtualizedGraphics.framework](https://developer.apple.com/documentation/paravirtualizedgraphics))、Linux用のデバイスドライバがありません。また、仮にLinux用のデバイスドライバがあったとしても、Linuxで動くMetal.frameworkがないこと、仮想GPU ↔ ホストGPU間のコマンド転送のシリアライズプロトコルが不明であること、などから、Virtualization.framework上のLinux仮想マシンでGPUを使えるようにするためには、かなりハードルが高そうです (実際、Virtualization.frameworkを使う仮想化ツールの中で、LinuxゲストでGPUが使えるものはまだないはずです)。

また、Virtualization.frameworkのvirtiofs実装にはいくつか課題があることがわかってきました。具体的には同時書き込みに関する制限や、SELinuxと組み合わせたときの動き等です。Virtualization.frameworkはAppleのプロプライエタリなソフトウェアであるため、修正するにはAppleに要望を出して結果を待つことしかできないのがつらいところです。

- [macOS vm virtiofs concurrency issue #23061](https://github.com/containers/podman/issues/23061)
- [VirtioFS causes intermittent failures with Rust #7059](https://github.com/docker/for-mac/issues/7059)

# libkrunという選択肢

libkrunというライブラリ形式のVMMがあります[^5]。Linux KVMだけでなく[macOSのHypervisor.frameworkのサポート](https://github.com/containers/libkrun/pull/161)が入っていること、virtio-gpuが使えること、OSSであること(メインの開発者はRed Hatの人です)、ということから、libkrunを使った仮想マシンでMacのホストGPUを使えるか検討することになりました。

[^5]: libkrunの詳細については https://logmi.jp/tech/articles/324735 https://rheb.hatenablog.com/entry/libkrun-intro をご参照ください。

::: details 余談 (libkrunでvirtio-gpuが使えるようになったきっかけは「Asahi Linux上でSteamゲームを実行したかったから」という小噺)
macOS用PodmanでのGPU利用方法を検討していた時点(2024年初め頃)で、libkrunには既にLinuxホスト上のマイクロVMでのvirtio-gpuが使えるようになっていたのですが、その理由は「Asahi Linux (Apple Silicon Mac用のLinuxディストリビューション) 上でSteamゲームを実行したかったから」です。

libkrun開発者の[ブログ記事](https://sinrega.org/2023-10-06-using-microvms-for-gaming-on-fedora-asahi/)に詳細が載っていますが、Asahi LinuxでSteamゲームを実行するために、libkrunに対して以下の機能追加を行いました。

- Asahi Linuxはページサイズが16KBだが、x86のゲームバイナリを動かすにはページサイズを4KBにする必要がある。一方で、 Apple Silicon搭載MacはIOMMUチップが16KBページで動くことが前提となっており[^6]、実質4KBページサイズのカーネルは(少なくとも現状では)動かない[^7]。この対応策として、Asahi Linux上でlibkrunを使って4KBページサイズのaarch64 LinuxのmicroVMを起動する
- x86コードの実行にはFEX-Emuを使う (FEX-Emuはaarch64上でx86コードを実行するためのエミュレータ、いろいろ最適化が施されており、Qemuよりも高速に実行できる、TSOサポートもあり)
- virtio-gpuの実装は[crosvm](https://github.com/google/crosvm)の[rutabaga_gfx](https://crosvm.dev/doc/rutabaga_gfx/index.html) crateを流用できそう ([PR](https://github.com/containers/libkrun/pull/144))
  - そのままだとゲームをするにはパフォーマンスが足りないので、AGX (Asahi LinuxのGPUドライバ)、Mesa、virglrendererを改良してDRM Native Context[^8]が使えるようにする
- Steamの実装上の事情で、TSOではダメでvirtio-netが必要になった(が既に実装済みだった) ([PR](https://github.com/containers/libkrun/pull/142))
  - 元々libkrunは、vsockとTSI (Transparent Socket Impersonation) という独自技術を組み合わせて、virtio-netを使わずにネットワーク通信を実現していました[^9]
- virtio-sndも実装しちゃえ ([PR](https://github.com/containers/libkrun/pull/186))

そんなこんなで、今ではAsahi Linux上で多くのSteamのゲームを楽しめるようになっています。

https://x.com/slpnix/status/1704413076320117096

https://x.com/LinaAsahi/status/1794408897572462669

https://x.com/LinaAsahi/status/1795777842653065623

このときに実装したvirtio-gpuを、mac用PodmanでGPUを利用するにあたって活用することができました。

(余談おわり)
:::

[^6]: 詳細は https://asahilinux.org/2021/10/progress-report-september-2021/#linux-drivers-galore の "IOMMU 4K patches" をご参照ください
[^7]: 対応パッチが提案されていますが、まだ取り込まれていないようです: [[PATCH v3 0/6] Support IOMMU page sizes larger than the CPU page size](htt5s://lore.kernel.org/linux-iommu/20211019163737.46269-1-sven@svenpeter.dev/)
[^8]: DRM Native ContextはGoogleが開発したvirtio-gpu高速化技術。https://indico.freedesktop.org/event/2/contributions/53/attachments/76/121/XDC2022_%20virtgpu%20drm%20native%20context.pdf 既存のAPI Remotingの手法 (VMからのOpenGLやVulkanのAPI呼び出しを全てホストに転送して描画する) ではオーバーヘッドが大きいが、DRM Native Contextでは実際にGPU操作を行うコマンドのみをホストに送ることで処理を効率化し、ほぼ物理アクセスに近い性能が出せるようになる。現状、QualcommのfreedrenoとAMDのamdgpuがDRM Native Contextに対応している
[^9]: TSIを使ったlibkrunのネットワーク通信については https://rheb.hatenablog.com/entry/libkrun-networking をご参照ください

# Podman Machineをlibkrunで動かすために

v5.0以降のPodmanは、[vfkit](https://github.com/crc-org/vfkit)という、Virtualization.frameworkのGo bindingである[vz](https://github.com/Code-Hex/vz)を使ったCLIを使って仮想マシンを起動します。

libkrunをPodman Machineのproviderにするにあたって、[krunkit](https://github.com/containers/krunkit)というlibkrunのwrapperとなるCLIを作りました。Podman側の変更を最小限に抑えるために、krunkitにはvfkitとほぼ同じコマンドラインオプションが実装されています。

元々libkrunは、BIOS/UEFI含めた完全な仮想ハードウェアを作るものではなく、必要最低限の機能を提供する軽量なVMMでした。また当初はLinux KVMを前提としていましたが、2021年に[Hypervisor.frameworkのサポートが加わり](https://github.com/containers/libkrun/pull/15)、macOS上でもマイクロVMを動かせるようになっていました。

macOS上でのLinuxゲストからのGPUアクセスをサポートするにあたり、

- [UEFIのサポート](https://github.com/containers/libkrun/pull/161)(現時点ではaarch64のみ)、[SMBIOS Type 11のOEM Stringを渡す](https://github.com/containers/libkrun/pull/187)こともできます
- [virtio-gpuのVenusプロトコルのサポート](https://github.com/containers/libkrun/pull/174)

を追加することにより、最終的にLinuxゲストからGPUを使えるようになりました。

Linuxゲスト上のアプリケーションがVulkan APIでvirtio-gpuにアクセスすると、Vulkanコマンドが[Venusプロトコル](https://docs.mesa3d.org/drivers/venus.html)でシリアライズされてVMMの[virgl](https://docs.mesa3d.org/drivers/virgl.html)[renderer](https://gitlab.freedesktop.org/virgl/virglrenderer)にわたり、さらに[MoltenVK](https://github.com/KhronosGroup/MoltenVK)を使って[Metal API](https://developer.apple.com/metal/)に変換してGPUハードウェアにアクセスします。

![](/images/virtio-gpu-acceleration-software-stack.png)
*Linuxゲストから仮想化GPUを使用するときのソフトウェアスタック*

# llama.cpp入りコンテナイメージ

本記事前半での動作確認は、`quay.io/slopezpa/fedora-vgpu-llama` をコンテナとして使用しましたが、同じようなことをするコンテナイメージを作ってみました。

https://github.com/orimanabu/fedora-llama-vulkan

中でやっているのは、ちょっと古いllama.cppをVulkanを使うようにコンパイルしているだけです。MesaのVulkanドライバは、メモリ確保のサイズを16KB単位になるようアラインメント調整する等、いくつかのパッチを当てたものを使っています[^10]。

# 参考文献

- DevConf.US 2024: [GPU Accelerated Containers on Apple Silicon with libkrun and podman machine](https://pretalx.com/devconf-us-2024/talk/YWQZ8X/)
- [Enabling containers to access the GPU on macOS](https://sinrega.org/2024-03-06-enabling-containers-gpu-macos/)
- [Podman and Libkrun](https://blog.podman.io/2024/07/podman-and-libkrun/)
- https://x.com/slpnix/status/1813482806040756598

[^10]: パッチの当たったMesaのrpmは[slp/mesa-krunkitのcopr](https://copr.fedorainfracloud.org/coprs/slp/mesa-krunkit/)にあります。パッチの内容は https://copr-dist-git.fedorainfracloud.org/cgit/slp/mesa-krunkit/mesa.git/tree/?h=f40 をご参照ください。

