---
title: "Podmanのコンテナイメージはファイル単位で重複排除ができるようになりました"
title: "pullしたコンテナイメージをファイル単位で重複排除するzstd:chunkedの紹介"
title: "コンテナイメージレイヤーをまたいでファイル単位で重複排除するイメージフォーマットzstd:chunkedの紹介"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに

本記事は、OpenShift Advent Calendar 2024の12/3の記事で、[containers/storage](https://github.com/containers/storage)ライブラリで実装されている新しいコンテナイメージのフォーマット `zstd:chunked` について紹介します。container/storageの機能なので、コンテナエンジン/ランタイムとしてはPodmanとかCRI-Oで使えます。本記事はPodmanでの例を挙げますが、CRI-O(およびCRI-Oを使っているOpenShift)でも同じことができます。

レイヤー単位の重複排除を意識してDockerfileを書く必要がなくなる (`--squash-all` してベースイメージ含めてひとつのレイヤーに押し込めても大丈夫)
過去にpull済みのファイルは再度取得する必要はない
複数レイヤーに存在する同一ファイルは、ひとつ分しかストレージ容量を消費しない
複数レイヤー(もしくはコンテナ)に存在する、read-onlyファイルは、メモリ上に一度しかmapされない

zstd:chunkedの「zstd ([Zstandard](https://facebook.github.io/zstd/))」は、ご存知の方もいらっしゃるかと思いますが、圧縮フォーマットのひとつです。Metaの人が中心となって開発し、RFC8478/8878で規格化されています。zstdは、圧縮率に関してはzipやgzipと比べると同等以上でxzと比べると多少悪い(圧縮レベルにもよりますが)、でも圧縮/展開のスピードは圧倒的に速い、という特徴を持ちます。FedoraやUbuntuなどのLinuxディストリビューションでは、パッケージの圧縮にも使われています。

zstdはコンテナイメージのメディアタイプとしても使うことができます。従来の `application/vnd.oci.image.layer.v1.tar+gzip` に加えて、zstdで圧縮する `application/vnd.oci.image.layer.v1.tar+zstd` が2019年8月にOCI Image Specで定義され[^1]、代表的なコンテナレジストリ、コンテナエンジン含む関連ツールで使用できます。
`application/zstd` のMIME Media TypeはOCI image specに入っており、zstd圧縮形式のコンテナイメージは今はほとんどのコンテナレジストリで使えるはずです(少なくともdocker.ioおよびquay.ioでは使えます)。

[^1]: https://github.com/opencontainers/image-spec/pull/788

従来のコンテナイメージにレイヤーは、ファイルシステムのツリー(のサブセット)をtarでまとめてgzipで圧縮したものです。
これを「個々のファイルをzstd(もしくはgzip)で圧縮してtarでまとめたもの」に変えて、最後に「そのtarballのどこにどんなファイルが入っているか」の目次(TOC, Table of Contents)の情報を追加したものが、zstd:chunkedです。

containerdのsnapshotter(コンテナで使用するファイルシステムのライフサイクルを管理するコンポーネント)のひとつにstarga-snapshotterがあり、そこでeStargzというイメージフォーマットが使われています。eStargzも「個々のファイルをgzipで圧縮してtarでまとめたもの」にTOC情報をつけたコンテナイメージフォーマットです。

eStargzもzstd:chunkedも、GoogleのCRFSというPoCプロジェクトで考案されたstargz (Seekable tar + gz) にインスパイアされたフォーマットで、「個々のファイルをzstd/gzipで圧縮したものを連結して、それを伸長すると、もとのファイル群を連結したものになる」という性質を使っています。この性質により、eStargzやzstd:chunkedに対応していないコンテナランタイムでも、image-specの `application/vnd.oci.image.layer.v1.tar+gzip` や `application/vnd.oci.image.layer.v1.tar+zstd` に対応していれば、通常の圧縮ファイルとしてイメージを扱うことができます。

eStargzはlazy pulling (ファイル単位で必要になったタイミングでpullする) が大きな特徴のひとつですが、containers/storageライブラリのzstd:chunked自体にはlazy pullingの機能はありません。

zstd:chunkedは、


[RFC8878の3.1 Frames](https://datatracker.ietf.org/doc/html/rfc8878#name-frames)に次のような記載があります: 

> Zstandard compressed data is made up of one or more frames. Each frame is independent and can be decompressed independently of other frames. The decompressed content of multiple concatenated frames is the concatenation of each frame's decompressed content.
> 
> There are two frame formats defined for Zstandard: Zstandard frames and skippable frames. Zstandard frames contain compressed data, while skippable frames contain custom user metadata.

つまりzstdには、gzipと同様、「個々のファイルをzstdで圧縮したものを連結して、それを伸長すると、もとのファイル群を連結したものになる」という性質があります。

従来のコンテナイメージにレイヤーは、ファイルシステムのツリー(のサブセット)をtarでまとめてgzipで圧縮したものです。これを「個々のファイルをzstd(もしくはgzip)で圧縮してtarでまとめたもの」にしても、



zstd:chunkedは `tar+zstd` メディアタイプの派生形で、

zstd:chunkedはそのvariantで、eStargzのようなファイル単位でのイメージ取得ができるようになります。

containers/storageライブラリのzstd:chunked設定はさらに一歩進んでおり、コンテナイメージを取得する際、
```
すでに取得しているコンテナイメージのインデックスを見て、SHA256 IDを比較してすでにpull済みであれば、そのファイルはpullせずに取得済みのファイルのハードリンクを作成、もしくは reflink copyを実施する
```
という処理を行います。

zstd:chunkedは新しいコンテナイメージレイヤーのフォーマットです。eStargzと同じく、[Google CRFS](https://github.com/google/crfs)をベースとして考案されました。またxxxなど、いくつかの考え方や実装はeStargzを参考にしています。

zstd:chunkedのメタデータをアポートしないコンテナランタイムにとっては、通常のzstd圧縮のレイヤーとして見えます。
zstdのレイヤーは、多くのコンテナ周辺ツールですでにサポートされています。
- Moby v23 (2023-02)
- containerd v1.5 (2021-05)
- containers/storage (2019-08)
- CRI-O (2020-02)
Fedoraでコンテナイメージのデフォルトをzstdにしようという提案2024年6月にありましたが、結局rejectされています。
https://fedoraproject.org/wiki/Changes/zstd:chunked
https://discussion.fedoraproject.org/t/switch-fedora-container-images-to-support-zstd-chunked-format-by-default/123712
https://discussion.fedoraproject.org/t/f41-change-proposal-default-podman-created-images-to-zstd-chunked-self-contained/125540

従来のgzipやzstdといったtarballを圧縮したイメージレイヤーでは、レイヤー単位でSHA256のハッシュ値を見て同じものかどうかを判断するため、異なるレイヤーに同じファイルが存在する可能性があります。zstd:chunkedでは、すでに同じものが手元のコンテナストレージにあるかどうかをファイル単位で判断することで、より効率的にイメージを管理することができます。

具体的には、
- 複数のレイヤーにまたがって存在するファイルは、ひとつ分しかストレージを消費しません (ハードリンク、もしくはファイルシステムのreflink機能で実現します)
  - ハードリンクを使う設定の場合、ページキャッシュも共有できるため、メモリ使用量も効率化できます
- レイヤーをダウンロードする際、すでに存在するファイルはダウンロードしません。

# ストレージの効率化

sotrage.confで `use_hard_links = "true"` 設定をしていればハードリンクを作成する
use_hard_links設定が"false"であれば、ファイルシステムがreflinkをサポートしていればreflinkを作成する
use_hard_links設定が"false"で、かつファイルシステムがreflinkをサポートしていなければ、ファイルのコピーを作成する (この場合はストレージ効率化は行われない)

# メモリの効率化

ハードリンクの場合は、i-nodeが同じなので、異なるレイヤーの同じファイルに関して、ページキャッシュはひとつ分ですむ
reflinkの場合はi-nodeが異なるため、異なるレイヤーの同じファイルに関してページキャッシュは共有できない (reflinkのCoWにより、ストレージ消費は1個分で済むものの)


# ネットワークの効率化

pull時の動きとしては、

- まずTOC (Table of Contents)情報をダウンロードする。TOCのオフセットはimage manifestのアノテーションに記載されている
- TOCにリストされたレイヤー内のファイルごとに、
  - そのファイルが手元にあれば、ハードリンク/reflink/コピーを作成する
  - 手元になければ、HTTP Range Requestを使ってダウンロードする


composefsは、現在のところ、コンテナストレージおよびOSTreeベースのOSイメージのリポジトリとしてのユースケースが考えられています。本記事では、コンテナストレージとしてのユースケースにフォーカスします。

コンテナの文脈では、composefsは「ファイル単位で重複排除ができるコンテナストレージ」を提供するファイルシステムと言えます。通常のコンテナイメージは、レイヤーごとにSHA256のハッシュ値を計算し、複数のコンテナイメージで同じレイヤーを使用する場合はそのレイヤーを共有することで、イメージレイヤーごとの重複排除を行います。composefsを使用すると、より粒度の細かいファイル単位での重複排除行うことができます (複数のレイヤーに同じバイナリが入っている場合、1個分のストレージ容量しか使いません)。

なんでOpenShiftとは関係ないcomposefsをAdvent Calendarのネタにしているかといいますと... OpenShiftのノードは「RHEL CoreOS」というLinuxディストリビューションを使っているのですが、そのアップストリームであるFedora CoreOSでは、実はcomposefsが使われています。というわけで、それほど遠くない将来、OpenShiftに入ってくる(かもしれない)機能[^1]ということで、ご容赦ください... 

[^1]: 外から見えるJIRAチケットもありました https://issues.redhat.com/browse/COS-2963

# reflinknのとき

```
$ podman system reset
```

# 設定

```ini
storage.conf
[storage.options]
pull_options = {enable_partial_images = "true", use_hard_links = "false", ostree_repos=""}

[storage.options.overlay]
# use_composefs = "false"


containers.conf
[engine]
#add_compression = ["gzip", "zstd", "zstd:chunked"]
#compression_format = "gzip"
#compression_level = 5
```

```
$ find ~/.local/share/containers/storage/overlay -type f
/home/ori/.local/share/containers/storage/overlay/.has-mount-program
```

```
$ podman pull quay.io/manabu.ori/myhttpd:c0
Trying to pull quay.io/manabu.ori/myhttpd:c0...
Getting image source signatures
Copying blob 4ca6d99209cf done  81.2MiB / 81.2MiB (skipped: 42.4KiB = 0.05%)
Copying config ddb744a7a7 done   |
Writing manifest to image destination
284abe58dc8218d85fa8c7d5b111149b8b57061fb2d89ca2130d44c70f6199bb
```

(xfs)
```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 58465342
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       3:     123108..    123111:      4:
   1:        4..     143:   39138060..  39138199:    140:     123112: last,eof
```

```
$ podman pull quay.io/manabu.ori/myhttpd:c1
Trying to pull quay.io/manabu.ori/myhttpd:c1...
Getting image source signatures
Copying blob 63df9f9dafda done  81.2MiB / 81.2MiB (skipped: 79.4MiB = 97.87%)
Copying config 0ca3f4f445 done   |
Writing manifest to image destination
9abebd0ec4a754fb65e2ab985c89b71b9fa339a23606e0288d6526e78a4d47dd
```

```
$ ls -i ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
241127 /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
$ ls -i ~/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
7309808 /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
```

```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 58465342
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       3:     123108..    123111:      4:             shared
   1:        4..     143:   39138060..  39138199:    140:     123112: last,shared,eof
/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls: 2 extents found
```


(btfs)
```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 9123683e
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..     127:    2438128..   2438255:    128:             encoded
   1:      128..     143:    2424244..   2424259:     16:    2438256: last,encoded,eof
/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls: 2 extents found
```

```
$ sudo btrfs filesystem du -s ~/.local/share/containers/storage/overlay/*
     Total   Exclusive  Set shared  Filename
 218.59MiB   218.59MiB       0.00B  /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/l
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/staging
```

```
$ podman pull quay.io/manabu.ori/myhttpd:c1
Trying to pull quay.io/manabu.ori/myhttpd:c1...
Getting image source signatures
Copying blob 63df9f9dafda done  81.2MiB / 81.2MiB (skipped: 79.4MiB = 97.87%)
Copying config 0ca3f4f445 done   |
Writing manifest to image destination
9abebd0ec4a754fb65e2ab985c89b71b9fa339a23606e0288d6526e78a4d47dd
```

```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 9123683e
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..     127:    2438128..   2438255:    128:             encoded,shared
   1:      128..     143:    2424244..   2424259:     16:    2438256: last,encoded,shared,eof
/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls: 2 extents found
```

```
$ sudo btrfs filesystem du -s ~/.local/share/containers/storage/overlay/*
     Total   Exclusive  Set shared  Filename
 218.59MiB    11.46MiB   120.48MiB  /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
 209.22MiB       0.00B   120.48MiB  /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/l
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/staging
```

# hardlinkのとき

```
$ sudo cp /usr/share/containers/storage.conf /etc/containers/
```

```
$ sudo vim /usr/share/containers/storage.conf
```

```
$ crudini --get /etc/containers/storage.conf storage.options pull_options
{enable_partial_images = "true", use_hard_links = "true", ostree_repos=""}
```

```
$ podman system reset -f
```

```
$ podman pull quay.io/manabu.ori/myhttpd:c0
Trying to pull quay.io/manabu.ori/myhttpd:c0...
Getting image source signatures
Copying blob 4ca6d99209cf done  81.2MiB / 81.2MiB (skipped: 42.4KiB = 0.05%)
Copying config ddb744a7a7 done   |
Writing manifest to image destination
284abe58dc8218d85fa8c7d5b111149b8b57061fb2d89ca2130d44c70f6199bb
```

```
$ sudo du -csh ~/.local/share/containers/storage/overlay/*
234M	/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
0	/home/ori/.local/share/containers/storage/overlay/l
0	/home/ori/.local/share/containers/storage/overlay/staging
234M	total
```

```
$ podman pull quay.io/manabu.ori/myhttpd:c1
Trying to pull quay.io/manabu.ori/myhttpd:c1...
Getting image source signatures
Copying blob 63df9f9dafda done  81.2MiB / 81.2MiB (skipped: 79.4MiB = 97.87%)
Copying config 0ca3f4f445 done   |
Writing manifest to image destination
9abebd0ec4a754fb65e2ab985c89b71b9fa339a23606e0288d6526e78a4d47dd
```

```
$ sudo du -csh ~/.local/share/containers/storage/overlay/*
234M	/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
11M	/home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130
0	/home/ori/.local/share/containers/storage/overlay/l
0	/home/ori/.local/share/containers/storage/overlay/staging
244M	total
```

```
$ ls -i ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
241122 /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
$ ls -i ~/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
241122 /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
```

# 10個

```
$ podman image ls
REPOSITORY                  TAG         IMAGE ID      CREATED     SIZE
quay.io/manabu.ori/myhttpd  c9          4e5d2e31e232  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c8          89bffbd7f7f2  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c7          9a2a54fe127c  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c6          294570b6402f  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c5          dc9f54791668  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c4          ce5114de2d6c  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c3          163518020f87  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c2          38fb1cd4cb8c  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c1          9abebd0ec4a7  3 days ago  227 MB
quay.io/manabu.ori/myhttpd  c0          284abe58dc82  3 days ago  229 MB
```

```
$ sudo du -csh ~/.local/share/containers/storage/overlay/*
230M	/home/ori/.local/share/containers/storage/overlay/00db55db807cab800fdf6b6ed39ec46e8ec53c1e4b755893f9f6d30d500fe4d0
15M	/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
11M	/home/ori/.local/share/containers/storage/overlay/30fec452bc3836be305803c342af9f11a5ca6e16f38496176b6c80bee168ae69
11M	/home/ori/.local/share/containers/storage/overlay/36fc922787f2a9132ba3d7ea9decc8db6203518c6e05a61a049b91f6c3c66f81
11M	/home/ori/.local/share/containers/storage/overlay/480110807cafc058472a3f2eea4d8f9a57bc0a528d6ae21108bf05475e4d28c7
11M	/home/ori/.local/share/containers/storage/overlay/79241936613eb087b6b84a3939bbbadcb0bb198b089c65c90e916f9b06688123
11M	/home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130
11M	/home/ori/.local/share/containers/storage/overlay/f1674671929ad75874ecd2bf1fc394ee394a1802acafcb899c7a703c194c7cb7
11M	/home/ori/.local/share/containers/storage/overlay/f401afca4b05f6a3dfa89dd0caf6f7cbc5c3a8706d1f42084ddabac370b7ba88
11M	/home/ori/.local/share/containers/storage/overlay/f72b3dfebd3aca217b7f0d5ef0a75e50567a696faafd058b1176cd00ef80f418
4.0K	/home/ori/.local/share/containers/storage/overlay/l
0	/home/ori/.local/share/containers/storage/overlay/staging
328M	total
```

(hardlink)
```
$ for i in $(seq 0 9); do ((port = i + 9000)); podman run --name myhttpd-c${i} --rm -d -p ${port}:80 quay.io/manabu.ori/myhttpd:c${i} sleep infinity;
done
```

```
$ smem -t -P '^sleep'
  PID User     Command                         Swap      USS      PSS      RSS
467575 ori      sleep infinity                     0       92      239     1556
467379 ori      sleep infinity                     0       96      243     1560
467477 ori      sleep infinity                     0       96      244     1560
467346 ori      sleep infinity                     0       96      253     1552
467411 ori      sleep infinity                     0       92      260     1640
467539 ori      sleep infinity                     0       96      264     1644
467508 ori      sleep infinity                     0       96      268     1624
467445 ori      sleep infinity                     0      100      272     1628
467314 ori      sleep infinity                     0       96      291     1708
467280 ori      sleep infinity                     0      120      304     1660
-------------------------------------------------------------------------------
   10 1                                           0      980     2638    16132
```

```
$ smem -t -P '^httpd -D'
  PID User     Command                         Swap      USS      PSS      RSS
460115 100047   httpd -D FOREGROUND                0      268      618     5252
460963 100047   httpd -D FOREGROUND                0      268      618     5220
462016 100047   httpd -D FOREGROUND                0      268      618     5280
460753 100047   httpd -D FOREGROUND                0      268      620     5364
461385 100047   httpd -D FOREGROUND                0      268      620     5360
461802 100047   httpd -D FOREGROUND                0      272      620     5248
461594 100047   httpd -D FOREGROUND                0      268      621     5412
460329 100047   httpd -D FOREGROUND                0      268      622     5440
461172 100047   httpd -D FOREGROUND                0      272      624     5360
460541 100047   httpd -D FOREGROUND                0      268      625     5420
460947 ori      httpd -D FOREGROUND                0      388     1179    11408
461785 ori      httpd -D FOREGROUND                0      392     1183    11404
460099 ori      httpd -D FOREGROUND                0      392     1184    11400
461577 ori      httpd -D FOREGROUND                0      400     1211    11508
461998 ori      httpd -D FOREGROUND                0      408     1221    11532
460525 ori      httpd -D FOREGROUND                0      436     1246    11672
460313 ori      httpd -D FOREGROUND                0      428     1248    11612
461157 ori      httpd -D FOREGROUND                0      428     1261    11660
461368 ori      httpd -D FOREGROUND                0      432     1262    11656
460737 ori      httpd -D FOREGROUND                0      440     1280    11852
461174 100047   httpd -D FOREGROUND                0     1076     1502     8232
461183 100047   httpd -D FOREGROUND                0     1076     1502     8232
461387 100047   httpd -D FOREGROUND                0     1080     1502     8168
461398 100047   httpd -D FOREGROUND                0     1080     1502     8168
460755 100047   httpd -D FOREGROUND                0     1076     1504     8264
460767 100047   httpd -D FOREGROUND                0     1076     1504     8264
460331 100047   httpd -D FOREGROUND                0     1080     1506     8192
460347 100047   httpd -D FOREGROUND                0     1080     1506     8192
461804 100047   httpd -D FOREGROUND                0     1080     1507     8164
462020 100047   httpd -D FOREGROUND                0     1080     1507     8212
460543 100047   httpd -D FOREGROUND                0     1080     1508     8264
460559 100047   httpd -D FOREGROUND                0     1080     1508     8264
460118 100047   httpd -D FOREGROUND                0     1080     1509     8216
460965 100047   httpd -D FOREGROUND                0     1080     1509     8200
461596 100047   httpd -D FOREGROUND                0     1080     1510     8220
462030 100047   httpd -D FOREGROUND                0     1080     1514     8216
461819 100047   httpd -D FOREGROUND                0     1080     1516     8168
461605 100047   httpd -D FOREGROUND                0     1080     1517     8224
460137 100047   httpd -D FOREGROUND                0     1080     1518     8220
460978 100047   httpd -D FOREGROUND                0     1080     1518     8204
461386 100047   httpd -D FOREGROUND                0     1304     1723     8344
461173 100047   httpd -D FOREGROUND                0     1300     1725     8468
460754 100047   httpd -D FOREGROUND                0     1300     1727     8504
460330 100047   httpd -D FOREGROUND                0     1304     1730     8424
461803 100047   httpd -D FOREGROUND                0     1304     1730     8404
462019 100047   httpd -D FOREGROUND                0     1304     1730     8452
460542 100047   httpd -D FOREGROUND                0     1304     1731     8504
460116 100047   httpd -D FOREGROUND                0     1304     1732     8456
460964 100047   httpd -D FOREGROUND                0     1304     1732     8440
461595 100047   httpd -D FOREGROUND                0     1304     1733     8460
-------------------------------------------------------------------------------
   50 2                                           0    41448    65943   417800
```

(reflink)
```
$ for i in $(seq 0 9); do ((port = i + 9000)); podman run --name myhttpd-c${i} --rm -d -p ${port}:80 quay.io/manabu.ori/myhttpd:c${i} sleep infinity;
done
```

```
$ smem -t -P '^sleep'
  PID User     Command                         Swap      USS      PSS      RSS
466252 ori      sleep infinity                     0     1420     1420     1428
466283 ori      sleep infinity                     0     1504     1504     1512
466153 ori      sleep infinity                     0     1508     1508     1516
466220 ori      sleep infinity                     0     1512     1512     1520
466314 ori      sleep infinity                     0     1512     1512     1520
466414 ori      sleep infinity                     0     1512     1512     1520
466189 ori      sleep infinity                     0     1596     1596     1604
466381 ori      sleep infinity                     0     1600     1600     1608
466121 ori      sleep infinity                     0     1640     1640     1648
466347 ori      sleep infinity                     0     1656     1656     1664
-------------------------------------------------------------------------------
   10 1                                           0    15460    15460    15540
```

```
$ smem -t -P '^httpd -D'
  PID User     Command                         Swap      USS      PSS      RSS
464380 100047   httpd -D FOREGROUND                0      272     1113     5028
463958 100047   httpd -D FOREGROUND                0      268     1128     5092
464601 100047   httpd -D FOREGROUND                0      268     1151     5208
465651 100047   httpd -D FOREGROUND                0      272     1152     5200
465439 100047   httpd -D FOREGROUND                0      268     1155     5220
464169 100047   httpd -D FOREGROUND                0      268     1165     5300
465015 100047   httpd -D FOREGROUND                0      268     1166     5280
463741 100047   httpd -D FOREGROUND                0      268     1171     5356
464802 100047   httpd -D FOREGROUND                0      272     1180     5300
465229 100047   httpd -D FOREGROUND                0      272     1200     5400
463960 100047   httpd -D FOREGROUND                0     1080     2389     7888
463970 100047   httpd -D FOREGROUND                0     1080     2399     7920
465232 100047   httpd -D FOREGROUND                0     1076     2399     8000
465240 100047   httpd -D FOREGROUND                0     1076     2399     8000
465653 100047   httpd -D FOREGROUND                0     1080     2401     7904
465017 100047   httpd -D FOREGROUND                0     1080     2404     7972
464392 100047   httpd -D FOREGROUND                0     1076     2410     7964
465441 100047   httpd -D FOREGROUND                0     1080     2413     8000
465023 100047   httpd -D FOREGROUND                0     1080     2414     7984
464382 100047   httpd -D FOREGROUND                0     1076     2416     7984
464804 100047   httpd -D FOREGROUND                0     1080     2420     8044
464811 100047   httpd -D FOREGROUND                0     1080     2420     8044
464604 100047   httpd -D FOREGROUND                0     1080     2422     8028
464171 100047   httpd -D FOREGROUND                0     1080     2424     8060
464180 100047   httpd -D FOREGROUND                0     1080     2432     8084
465447 100047   httpd -D FOREGROUND                0     1080     2434     8040
464613 100047   httpd -D FOREGROUND                0     1080     2445     8096
463743 100047   httpd -D FOREGROUND                0     1080     2447     8176
463759 100047   httpd -D FOREGROUND                0     1080     2447     8176
465659 100047   httpd -D FOREGROUND                0     1080     2455     8056
465652 100047   httpd -D FOREGROUND                0     1304     2615     8084
463959 100047   httpd -D FOREGROUND                0     1304     2620     8120
464381 100047   httpd -D FOREGROUND                0     1300     2634     8176
465230 100047   httpd -D FOREGROUND                0     1300     2634     8236
465440 100047   httpd -D FOREGROUND                0     1304     2645     8236
464170 100047   httpd -D FOREGROUND                0     1304     2652     8284
464603 100047   httpd -D FOREGROUND                0     1304     2654     8260
465016 100047   httpd -D FOREGROUND                0     1304     2654     8244
464803 100047   httpd -D FOREGROUND                0     1304     2655     8280
463742 100047   httpd -D FOREGROUND                0     1304     2680     8412
465628 ori      httpd -D FOREGROUND                0     3720     5165    10748
465418 ori      httpd -D FOREGROUND                0     3744     5177    10788
464574 ori      httpd -D FOREGROUND                0     3792     5215    10832
464996 ori      httpd -D FOREGROUND                0     3872     5256    10780
464784 ori      httpd -D FOREGROUND                0     3972     5360    10892
464360 ori      httpd -D FOREGROUND                0     3980     5371    10896
464150 ori      httpd -D FOREGROUND                0     4024     5415    10996
465206 ori      httpd -D FOREGROUND                0     4040     5423    10952
463938 ori      httpd -D FOREGROUND                0     4064     5450    10948
463724 ori      httpd -D FOREGROUND                0     4532     5941    11624
-------------------------------------------------------------------------------
   50 2                                           0    77052   140187   404592
```