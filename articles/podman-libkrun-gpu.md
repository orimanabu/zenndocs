---
title: "PodmanのコンテナからmacOSのGPUを使ってAIワークロードを高速処理できるようになりました"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# モチベーション

macOS上で、PodmanコンテナからGPUを使いたい！

# 前提知識

- macOS上でPodmanでコンテナを実行するには、Linux仮想マシン (Podman Machine) が必要
  - v5.0より前: Qemu
  - v5.0以降: Virtualization.framework

# 課題

## ゲストに対するGPUサポート

https://github.com/utmapp/UTM/discussions/5482



## Qemuの課題

安定性に問題、すでに捨てた
TSO/Rosettaサポート
virtiofsがないので、virtio-9pを使うしかない、遅いしSELinuxサポートに課題

## Virtualization.frameworkの課題

準仮想化GPUはある、がLinux用のデバイスドライバがない

Metal.frameworkがLinuxにない

# libkrun

libkrunというライブラリ形式のVMMがあります。Linux KVMだけでなく[macOSのHypervisor.frameworkのサポート](https://github.com/containers/libkrun/pull/161)が入っていること、virtio-gpuが使えること、OSSであること(メインの開発者はRed Hatの人です)、ということから、libkrunを使った仮想マシンでホストGPUを使えるか検討することになりました。

余談ですが、libkrunでvirtio-gpuが使えるようになったのは、「Asahi Linux上でSteamゲームを実行したかったから」です。

- Asahi Linuxはページサイズが4KBだが、x86_64のゲームバイナリを動かすにはページサイズを4KBにする必要がある。一方で、 Apple Silicon搭載MacはIOMMUチップが16KBページで動くことが前提となっており[^2]、実質4KBカーネルは(現状では)動かない。この対応策として、Asahi Linux上でlibkrunを使って4KBページサイズのaarch64 LinuxのmicroVMを起動する
- x86コードの実行はFEX-Emuを使う
- virtio-gpuの実装は[crosvm](https://github.com/google/crosvm)の[rutabaga_gfx](https://crosvm.dev/doc/rutabaga_gfx/index.html) crateを流用できそう ([PR](https://github.com/containers/libkrun/pull/144))
  - そのままだとゲームをするにはパフォーマンスが足りないので、AGX (Asahi LinuxのGPUドライバ)、Mesa、virglrendererを改良してDRM native context[^3]が使えるようにするぜ
- Steamの実装上の事情で、virtio-netが必要になった(が既に実装済みだった) ([PR](https://github.com/containers/libkrun/pull/142))
  - 元々libkrunは、vsockとTSI (Transparent Socket Impersonation) という独自技術を組み合わせて、virtio-netを使わずにネットワーク通信を実現していました[^4]
- virtio-sndも実装しちゃえ ([PR](https://github.com/containers/libkrun/pull/186))

https://x.com/slpnix/status/1704413076320117096

https://x.com/LinaAsahi/status/1794408897572462669

https://x.com/LinaAsahi/status/1795777842653065623

[^2]: 詳細は https://asahilinux.org/2021/10/progress-report-september-2021/#linux-drivers-galore の "IOMMU 4K patches" をご参照ください
[^3]: DRM Native ContextはGoogleが開発したvirtio-gpu高速化技術。https://indico.freedesktop.org/event/2/contributions/53/attachments/76/121/XDC2022_%20virtgpu%20drm%20native%20context.pdf 既存のAPI Remotingの手法 (VMからのOpenGLやVulkanのAPI呼び出しを全てホストに転送して描画する) ではオーバーヘッドが大きいが、DRM Native Contextでは実際にGPU操作を行うコマンドのみをホストに送ることで処理を効率化し、ほぼ物理アクセスに近い性能が出せるようになる。現状、QualcommのfreedrenoとAMDのamdgpuがDRM native contextに対応している
[^4]: 詳細は https://rheb.hatenablog.com/entry/libkrun-networking をご参照ください

# Podman Machineをlibkrunで動かすために

Podmanはv5.0から、VMMとして[Virtualization.framework](https://developer.apple.com/documentation/virtualization)を使っています (`provider = applehv`)。具体的には、[vfkit](https://github.com/crc-org/vfkit)という、Virtualization.frameworkのGo bindingである[vz](https://github.com/Code-Hex/vz)を使ったCLIを使って仮想マシンを起動します。

libkrunをPodman Machineのproviderにするにあたって、[krunkit](https://github.com/containers/krunkit)というlibkrunのwrapperとなるCLIを作りました。Podman側の変更を最小限に抑えるために、krunkitにはvfkitとほぼ同じコマンドラインオプションが実装されています。

元々libkrunは、BIOS/UEFI含めた完全な仮想ハードウェアを作るものではなく、必要最低限の機能を提供する軽量なVMMでした。また当初はLinux KVMを前提としていました。その後着々と進化を続け、特に

- ハイパーバイザーとしてLinux KVMの他にApple Hypervisor.frameworkもサポートします (つまりmacOS上でも稼働します) [^1]
- UEFIをサポートします(現時点ではaarch64のみ)。SMBIOS Type 11のOEM Stringを渡すこともできます [^2]

の2点の進化により、macOS上でのサポートが強化されました。

libkrunはコンパイルオプションを変えることで2種類のバリアントがあります。
- libkrun: 普通の仮想マシンを管理するVMM
- libkrun-sev: AMD SEV対応のVMM (メモリ暗号化、リモートアテステーション)



サポートされる準仮想化デバイス
- virtio-console
- virtio-vsock
- virtio-fs
- virtio-baloon
- virtio-rng

# Internal
- [Containerization Guild | GPU Accelerated Containers w/ libkrun and podman machine](https://docs.google.com/presentation/d/1jwkivMU1eWCy0unoSOK6Na0wb_KMUn4WGCQquVxt2tA/edit?usp=sharing)
- [Trying out krunkit (GPU acceleration for Linux VMs on macOS)](https://docs.google.com/document/d/1IZCWAY5zMHqd0YlbnpGtCe7HNeWKQNHi8RuhujAJmg0/edit?usp=sharing)

# 重要
- DevConf.US 2024: [GPU Accelerated Containers on Apple Silicon with libkrun and podman machine](https://pretalx.com/devconf-us-2024/talk/YWQZ8X/)

- [The Linux graphics stack in a nutshell, part 1](https://lwn.net/Articles/955376/)
- [The Linux graphics stack in a nutshell, part 2](https://lwn.net/Articles/955708/)

- gist: [Linux Desktop on Apple Silicon in Practice](https://gist.github.com/akihikodaki/87df4149e7ca87f18dc56807ec5a1bc5)

- [GEM v. TTM](https://lwn.net/Articles/283793/)
- [GEM - the Graphics Execution Manager](https://lwn.net/Articles/283798/)

# Qemu

## TSO patch
- [[PATCH 0/4] arm64: Support the TSO memory model](https://lore.kernel.org/all/20240411-tso-v1-0-754f11abfbff@marcan.st/)
- [[PATCH 0/3] tso: aarch64: Expose TSO for virtualized linux on Apple Silicon](https://lore.kernel.org/lkml/20240410211652.16640-1-zayd_qumsieh@apple.com/)

- [[PATCH v6 00/11] Support blob memory and venus on qemu](https://patchew.org/Xen/20231219075320.165227-1-ray.huang@amd.com/)
- [[PATCH v2 10/12] hw/vmapple/apple-gfx: Introduce ParavirtualizedGraphics.Framework support](https://patchew.org/QEMU/20230830161425.91946-1-graf@amazon.com/20230830161425.91946-11-graf@amazon.com/)

- [3D accelerated qemu on MacOS](https://github.com/knazarov/homebrew-qemu-virgl)

- [virtio-gpu](https://www.qemu.org/docs/master/system/devices/virtio-gpu.html)

## virtio-fs
- [Add macOS support](https://gitlab.com/virtio-fs/virtiofsd/-/issues/169)

# Mesa
- [asahi: Sep 5](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25052)

# libkrun
- [virtio/fs/macos: overhaul to use macos inodes #176](https://github.com/containers/libkrun/pull/176)
- [Implement an EFI flavor #161](https://github.com/containers/libkrun/pull/161)
- [Extend virtio-gpu to support Venus on macOS #174](https://github.com/containers/libkrun/pull/174)
- [virtio-net implementation using passt #142](https://github.com/containers/libkrun/pull/142)
- [Import rutabaga_gfx+virtio_gpu from crosvm #144](https://github.com/containers/libkrun/pull/144)
- 2022-11: [macos: add support for running x86 microVMs on ARM64 #88](https://github.com/containers/libkrun/pull/88)

# Podman
- [Set applehv as default darwin provider #21145](https://github.com/containers/podman/pull/21145#issuecomment-1913717227)

# GPU virtualization

- Collabora blog 2018-05: [GPU virtualization update](https://www.collabora.com/news-and-blog/blog/2018/05/09/gpu-virtualization-update/)
- Collabora blog 2018-09: [Virtme: The kernel developers' best friend](https://www.collabora.com/news-and-blog/blog/2018/09/18/virtme-the-kernel-developers-best-friend/)
- Collabora blog 2018-02: [Virtualizing GPU Access](https://www.collabora.com/news-and-blog/blog/2018/02/12/virtualizing-gpu-access/)
- Collabora blog 2021-11: [Venus on QEMU: Enabling the new virtual Vulkan driver](https://www.collabora.com/news-and-blog/blog/2021/11/26/venus-on-qemu-enabling-new-virtual-vulkan-driver/)

- Phoronix 2021-04: [Google's VirtIO-GPU "Venus" Vulkan Driver Merged Into Mesa 21.1](https://www.phoronix.com/news/VirtIO-GPU-Venus-Vulkan)
- Phoronix 2017-03: [QEMU Is Interested In Vulkan Guest Support "Vulkan-ize Virgl"](https://www.phoronix.com/news/QEMU-Possible-Vulkan)
- Phoronix 2023-06: [Mesa's Venus VirtIO-GPU Driver Adds More Extensions To Help Zink](https://www.phoronix.com/news/Mesa-Venus-Zink)
- Phoronix 2021-09: [Linux 5.16 To Feature More Extensible VirtIO GPU Driver With "Context Types" Addition](https://www.phoronix.com/news/VirtIO-Linux-5.16-Ctx-Type)
- Phoronix 2021-11: [Getting Experimental Vulkan Within QEMU VMs Using Linux 5.16+ Paired With Mesa's Venus](https://www.phoronix.com/news/Vulkan-Venus-QEMU-Guests)
- Phoronix 2024-05: [Mesa's Venus Vulkan Driver Updated To Allow QEMU Support](https://www.phoronix.com/news/Venus-Driver-QEMU-Support)

# GPU container app on Docker on Apple Silicon Mac
- [How to run Tensorflow in Docker on an Apple Silicon Mac](https://schultz-christian.medium.com/how-to-run-tensorflow-in-docker-on-an-apple-silicon-mac-c56f3127b696)
- [Apple Silicon GPUs, Docker and Ollama: Pick two.](https://chariotsolutions.com/blog/post/apple-silicon-gpus-docker-and-ollama-pick-two/)

# DRM Native Context
- [amdgpu: native context support for virtio](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/21658)
- [freedreno: virtgpu support for freedreno in VM guest](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/14900)
- [virtgpu-native-context: Add msm native context](https://gitlab.freedesktop.org/virgl/virglrenderer/-/merge_requests/693)


# slpnix's blog

- [Enabling containers to access the GPU on macOS](https://sinrega.org/2024-03-06-enabling-containers-gpu-macos/)
- [Using microVMs for Gaming on Fedora Asahi](https://sinrega.org/2023-10-06-using-microvms-for-gaming-on-fedora-asahi/)