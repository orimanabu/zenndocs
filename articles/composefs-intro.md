---
title: "コンテナストレージに特化したファイルシステムComposeFSとは"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに

本記事は、OpenShift Advent Calendar 2024の12/3の記事で、composefsというコンテナ環境向けのファイルシステムを紹介します。

composefsは、現在のところ、コンテナストレージおよびOSTreeベースのOSイメージのリポジトリとしてのユースケースが考えられています。本記事では、コンテナストレージとしてのユースケースにフォーカスします。

コンテナの文脈では、composefsは「ファイル単位で重複排除ができるコンテナストレージ」を提供するファイルシステムと言えます。通常のコンテナイメージは、レイヤーごとにSHA256のハッシュ値を計算し、複数のコンテナイメージで同じレイヤーを使用する場合はそのレイヤーを共有します。言い換えると、イメージレイヤーごとの重複排除を行います。composefsを使用すると、より粒度の細かいファイル単位で行うことができます。

なんでOpenShiftとは関係ないcomposefsをAdvent Calendarのネタにしているかといいますと... OpenShiftのノードは「RHEL CoreOS」というLinuxディストリビューションを使っているのですが、そのアップストリームであるFedora CoreOSでは、実はcomposefsが使われています。というわけで、それほど遠くない将来、OpenShiftに入ってくる(かもしれない)機能ということで、ご容赦ください...

# composefsの機能

composefsは、いくつかの特徴を持つ新しいファイルシステムです。

典型的なファイルシステムはブロックデバイスをバックエンドに持ちますが、対してcomposefsは通常のread-onlyのファイルの集合をバックエンドに持ちます。別の言い方をすると、「バックエンドとなる通常のファイルの集合 (＋メタデータの情報) に対してcomposefsマウントする」という、ある種のループバックマウントをするファイルシステムです。

バックエンドとなるファイル群は、「content-addressed backing files」と呼ばれます。content addressedという言葉は、ファイルの内容からファイルを指定できる、という気持ちが込められています。具体的には、バックエンドのファイルは、ファイルの中身のSHA256のハッシュ値がファイル名となります。ネットワーク機器には搭載されているCAM (Content Addressable Memory, 連想メモリ) が想起される言葉です。MACテーブルやルーティングテーブルの検索をハードウェアで高速処理する。

composefsは、fs-verityを使ってファイルシステム全体の改ざん検知ができます。fs-verityはLinuxカーネルが持つファイル単位の改ざん検知を行う仕組みです。fs-verity自体は、ファイルの中身にしか関与しません。つまりfs-verytyではメタデータの変更は検知できません。しかしcomposefsの場合は、ファイルシステムのメタデータ/ディレクトリツリーをEROFSのイメージとして持つため、このイメージファイルに対してもfs-verityのダイジェストを確認することで、ファイルシステム全体の改ざん検知ができます。

# 歴史

最初は、新規の独自ファイルシステムという形で、2022年11月に最初のRFC patchがLKMLに投稿されました。この後議論を重ねながらv2, v3とパッチが更新されますが、どちらかというとあまりカーネルコミュニティからは賛同は得られませんでした。むしろ提案されたユースケースであれば、新しいファイルシステムを作るよりも、似た機能を持つ既存の仕組み (overlayfs、erofs等) を改良する方がよいのではないか、という意見が出ました。

- [[PATCH RFC 0/6] Composefs: an opportunistically sharing verified image filesystem](https://lore.kernel.org/lkml/cover.1669631086.git.alexl@redhat.com/)
- [[PATCH v2 0/6] Composefs: an opportunistically sharing verified image filesystem](https://lore.kernel.org/lkml/cover.1673623253.git.alexl@redhat.com/)
- [[PATCH v3 0/6] Composefs: an opportunistically sharing verified image filesystem](https://lore.kernel.org/lkml/cover.1674227308.git.alexl@redhat.com/)

その後、2023年のLSFMM/BPF Summitを経て、composefsはカーネル内の新規のファイルシステムではなく、「メタデータやディレクトリツリーをerofsのイメージとし、ファイルデータをcontent-addressedなオブジェクトファイルとして、それらをoverlayfsで組み合わせる」という方向に方針転換することになりました。

その後、CopmoseFSを実現するためにいくつかの機能がoverlayfsに追加されました。代表的なものとしては
- overlayfsでmetadata only layerとdata only lower layerを持てるようにする https://lore.kernel.org/all/20230427130539.2798797-1-amir73il@gmail.com/
- overlayfsでfs-verityのサポート https://lore.kernel.org/linux-unionfs/cover.1687345663.git.alexl@redhat.com/
があります。

大まかな流れは、LWNの以下の記事を順に読むとわかりやすいかもしれません。

- [Composefs for integrity protection and data sharing](https://lwn.net/Articles/917097/)
- [Debating composefs](https://lwn.net/Articles/922851/)
- [A decision on composefs](https://lwn.net/Articles/933616/)

# 参考文献

- composefsのリポジトリ
  - https://github.com/containers/composefs
- Alexander Larsson (composefsの主要開発者の一人、LKMLにパッチを投げた人) のブログ記事
  - [Using Composefs in OSTree](https://blogs.gnome.org/alexl/2022/06/02/using-composefs-in-ostree/)
  - [Composefs state of the union](https://blogs.gnome.org/alexl/2023/07/11/composefs-state-of-the-union/)
  - [Announcing composefs 1.0](https://blogs.gnome.org/alexl/2023/09/26/announcing-composefs-1-0/)
- Giuseppe Scrivano (composefsの開発者の一人、LKMLでの議論に参加している人です) のブログ記事
  - [composefs - a file system for container images](https://www.scrivano.org/posts/2021-10-26-compose-fs/)

# xxx

Red Hatは最近OSのファイルシステム自体をコンテナイメージとして管理/配布しようとしていて、そこでもcomposefsの特徴が活かせるのでは、ということで、bootcやostreeとcomposefsの連携もいろいろ画策されています。

RHEL/FedoraのImage modeにおけるコアコンポーネントであるbootcは、upstreamでは既にデフォルトでcomposefsを使用するようになっています。https://github.com/containers/bootc/pull/304

ostreeは既にcontent addressableなリポジトリを
https://github.com/ostreedev/ostree/issues/2867


```
$ sudo podman run --name web --rm -d -p 8080:80 quay.io/manabu.ori/myhttpd
9392c9fbe31d7ce8012d316b5b662b79c31953173bc03358c01bb693c2ba0f10
```


# Linux kernel

44dfd84a6d54a675e35ab618d9fab47b36cb78cd
teach move_mount(2) to work with OPEN_TREE_CLONE

2db154b3ea8e14b04fee23e3fdfd5e9d17fbc6ae
vfs: syscall: Add move_mount(2) to move mounts around

a07b20004793d8926f78d63eb5980559f7813404
vfs: syscall: Add open_tree(2) to reference or clone a mount

v1:
https://lore.kernel.org/linux-unionfs/20230408164302.1392694-1-amir73il@gmail.com/
https://lore.kernel.org/linux-unionfs/20230412135412.1684197-1-amir73il@gmail.com/

v2:
https://lore.kernel.org/all/20230427130539.2798797-1-amir73il@gmail.com/

37ebf056d6cfc6638c821386bc33a9602bbc0d28
ovl: introduce data-only lower layers

989974c804574d250ac92d44e220081959ac8ac1
ovl: Enable metadata only feature

ae8cba4033bc16e8a07792428a48a50710cc0f3c
ovl: Add framework for verity support

https://lore.kernel.org/linux-unionfs/cover.1687345663.git.alexl@redhat.com/

184996e92e86c4a4224dc4aaee75b2ccd04b6e78
ovl: Validate verity xattr when resolving lowerdata

2d2f2d7322ff43e0fe92bf8cccdc0b09449bf2e1
ovl: user xattr

459c7c565ac36ba09ffbf24231147f408fde4203
ovl: unprivieged mounts

## ???

d47748e5ae5af6572e520cc9767bbe70c22ea498
ovl: automatically enable redirect_dir on metacopy=on

4120fe64dce4f73d1a595253568d9f27674f2071
ovl: Set redirect on upper inode when it is linked

d5791044d2e5749ef4de84161cec5532e2111540
ovl: Provide a mount option metacopy=on/off for metadata copyup

9cec54c83a8baba3099bb8b445a735b93ab9511f
ovl: Initialize ovl_inode->redirect in ovl_get_inode()

# Phoronix

## EROFS

- [EROFS Lands FSCache-Based Shared Domain Support For Linux 6.1](https://www.phoronix.com/news/Linux-6.1-EROFS)
- [EROFS Gets Low-Latency Decompression For Much Better Performance](https://www.phoronix.com/news/Linux-6.3-EROFS-Faster)
- [EROFS Receives Some Useful Improvements With Linux 6.4](https://www.phoronix.com/news/Linux-6.4-EROFS)
- [EROFS File-System Adding DEFLATE Compression Support](https://www.phoronix.com/news/EROFS-DEFLATE-Coming)
- [EROFS Lands DEFLATE Compression, F2FS Improves Zoned Devices In Linux 6.6](https://www.phoronix.com/news/EROFS-F2FS-Linux-6.6)
- [EROFS No Longer Treating MicroLZMA Compression As Experimental](https://www.phoronix.com/news/EROFS-MicroLZMA-Stable)

## composefs

- [Red Hat Developers Announce Work On New "Composefs" File-System](https://www.phoronix.com/news/Composefs)

## Bcachefs

- [Bcachefs Submitted For Review - Next-Gen CoW File-System Aims For Mainline](https://www.phoronix.com/news/Bcachefs-For-Review-Linux)
- [It's Looking Like Bcachefs Won't Be Merged For Linux 6.5](https://www.phoronix.com/news/Linux-6.5-Bcachefs-Unlikely)
- [Bcachefs File-System Plans To Try Again To Land In Linux 6.6](https://www.phoronix.com/news/Bcachefs-Plans-For-Linux-6.6)
- [Linus Torvalds Reviews The Bcachefs File-System Code](https://www.phoronix.com/news/Linux-Torvalds-Bcachefs-Review)
- [Bcachefs File-System Re-Submitted For Linux 6.6](https://www.phoronix.com/news/Bcachefs-For-Linux-6.6-PR)
- [Linus Torvalds Comments On Bcachefs Prospects For Linux 6.6](https://www.phoronix.com/news/Linus-Comments-Bcachefs-6.6)
- [Bcachefs Looks Like It Won't Make It For Linux 6.6](https://www.phoronix.com/news/Bcachefs-Delayed-Linux-6.6)
- [Bcachefs Pull Request Submitted For Linux 6.7](https://www.phoronix.com/news/Bcachefs-Tries-For-Linux-6.7)
- [Bcachefs Merged Into The Linux 6.7 Kernel](https://www.phoronix.com/news/Bcachefs-Merged-Linux-6.7)

## PuzzleFS

- [Cisco Posts Rust-Written PuzzleFS File-System Driver For Linux](https://www.phoronix.com/news/Experimental-Rust-PuzzleFS)
- [PuzzleFS Continues Striving To Be The Best File-System For Containers](https://www.phoronix.com/news/PuzzleFS-Development-Continues)

## initoverlayfs?

- [Red Hat Looks For Feedback On Its New Initoverlayfs File-System Proposal](https://www.phoronix.com/news/Red-Hat-Initoverlayfs)

# zstd:chunked

- [Pull container images faster with partial pulls](https://www.redhat.com/sysadmin/faster-container-image-pulls)
- Container Plumbling Days 2021: [Starting up Containers Super Fast With Lazy Pulling of Images](https://www.slideshare.net/slideshow/starting-up-containers-super-fast-with-lazy-pulling-of-images/244154126)

# misc
- Fosdem 2024: [Composefs and containers](https://fosdem.org/2024/schedule/event/fosdem-2024-3250-composefs-and-containers/)


- Podmanでcomposefsが使えるのはrootfulのみ (rootlessでは使えない)
https://github.com/containers/storage/blob/v1.55.0/drivers/overlay/overlay.go#L383





`I see EROFS as a block device that uses fscache, and composefs as an overlay for files instead of directories.`
https://lore.kernel.org/lkml/878rhvg8ru.fsf@redhat.com/

全てのファイルシステムがfs-verifyをサポートするわけではない

userspaceでマウントするのはリスクが高い、増やしたくない
At least Christian said, "FUSE and Overlayfs are adventurous enough and they don't have their own on-disk format."
https://lore.kernel.org/lkml/3ae1205a-b666-3211-e649-ad402c69e724@linux.alibaba.com/

