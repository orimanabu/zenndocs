---
title: "ファイル単位で重複排除できるコンテナイメージフォーマットzstd:chunked"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに

本記事は、[OpenShift Advent Calendar 2024](https://qiita.com/advent-calendar/2024/openshift)の12/3の記事で、[containers/storage](https://github.com/containers/storage)ライブラリで実装されている新しいコンテナイメージのフォーマット `zstd:chunked` について紹介します。container/storageの機能なので、コンテナエンジン/ランタイムとしてはPodmanとかCRI-Oで利用できます。本記事ではPodmanでの使用例を挙げますが、CRI-O(およびCRI-Oを使っているOpenShift)でも同じことができます。

コンテナイメージはレイヤー構造になっており、各レイヤーは、コンテナで使うファイルシステムツリーをtarでまとめてgzipで圧縮したものです。コンテナイメージをpullするときは、各レイヤーのSHA256のハッシュ値を見て、すでに取得済みであればそれを使い、手元になければコンテナレジストリからダウンロードします。コンテナイメージを扱うに当たって、いくつかの課題点が挙げられます。

- ネットワーク負荷: サイズの大きいレイヤーをダウンロードするには時間がかかります。レイヤーに存在するファイルの一部をすでに持っていたとしても、ダウンロードはレイヤー単位で行う必要があります。例えば脆弱性対応等の事情でレイヤーの一部だけ更新した場合でも、取得するにはレイヤー全体をダウンロードする必要があります。
- ストレージ使用量: 重複排除の単位はレイヤーであるため、同一のファイルが異なる複数のレイヤーに存在する場合があります。
- メモリ使用量: 中身が同じファイルが異なるレイヤーに存在する場合、カーネルがそれらをロードした際、別々にマッピングするため、余分なメモリを消費します。

zstd:chunkedを使うと、これらの課題に対して以下のようにいい感じに対処することができます。

- ネットワーク負荷 → zstd:chunkedでは、レイヤー単位ではなくレイヤー内のファイル(もしくは一定サイズのチャンク)単位でダウンロードできるようになるため、すでにダウンロード済みのファイルをスキップすることでダウンロード時間を短縮できます。
- ストレージ使用量 → 複数レイヤーに存在する同一内容のファイルは、ハードリンクもしくはreflink[^1]によってひとつ分しかストレージを消費しません (ハードリンクを使う場合は設定が必要でかつ注意が必要です(詳しくは後述)、reflinkが使えるかはファイルシステムによります、どちらも使えない場合は通常のファイルコピーをします)
- メモリ使用量: レイヤーをまたがって存在するファイルをハードリンクにした場合、カーネルは、それらが同一ファイルであるとわかるため、メモリマッピングは一度で済みます (ただしハードリンクを使う場合は注意が必要です、詳しくは後述)。

FedoraのPodmanではもうzstd:chunkedが使えます[^2]。RHEL 9.5[^3]および10.0 Beta[^4]でも、TechPreviewですがPodmanでzstd:chunkedが使えるようになりました。

[^1]: reflinkとは、複数のファイルで同じデータブロックを共有させるファイルシステムの機能です。この機能により、ファイルのコピーを高速に(一瞬で)行うことができます。ファイルのデータを更新した場合は、ファイルシステムのCoW (Copy on Write)の機能を使って変更部分だけを別ブロックに書きます。reflinkをサポートする代表的なファイルシステムはbtrfsおよびxfsです。そう言えば、最近coreutilsのcpが[デフォルトでreflinkを使えるか試す](https://git.savannah.gnu.org/cgit/coreutils.git/commit/?id=25725f9d41735d176d73a757430739fb71c7d043)ようになりましたね

[^2]: zstd:chunkedをデフォルト設定にするかどうかで二転三転してますが... https://fedoraproject.org/wiki/Changes/zstd:chunked

[^3]: RHEL 9.5 Release Notes: [Pushing and pulling images compressed with zstd:chunked is available as a Technology Preview](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/9.5_release_notes/index#Jira-RHEL-32267)

[^4]: RHEL 10.0 Beta Release Notes: [Pushing and pulling images compressed with zstd:chunked is available as a Technology Preview](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10-beta/html-single/10.0_beta_release_notes/index#Jira-RHEL-32266)

# zstdとは

zstd:chunkedの「zstd ([Zstandard](https://facebook.github.io/zstd/))」は、圧縮フォーマットのひとつです。Metaの人が中心となって開発し、RFC8478/8878で規格化されています。zstdは、圧縮効率に関してはzipやgzipと比べると同等以上でxzと比べると多少悪い(圧縮レベルにもよりますが)、でも圧縮/展開のスピードは圧倒的に速い、という特徴を持ちます。FedoraやUbuntuなど、パッケージの圧縮アルゴリズムにzstdを採用しているのLinuxディストリビューションもあります。

![](https://raw.githubusercontent.com/facebook/zstd/v1.3.4/doc/images/CSpeed2.png)
*Compression Speed vs Ratio (cited from https://facebook.github.io/zstd/)*

![](https://raw.githubusercontent.com/facebook/zstd/v1.3.4/doc/images/DSpeed3.png)
*Decompression Speed (cited from https://facebook.github.io/zstd/)*

OCI Image Specにおいて、従来の `application/vnd.oci.image.layer.v1.tar+gzip` に加えて、zstdで圧縮する `application/vnd.oci.image.layer.v1.tar+zstd` がコンテナイメージのメディアタイプとして2019年8月に追加されました[^5]。それにともない、代表的なコンテナエンジン/ランタイムは下記バージョンからzstdをサポートしています。

- Moby v23 (2023-02)
- containerd v1.5 (2021-05)
- containers/storage (2019-08)
- containers/image v4.0.0 (2019-10)
- Podman v1.7.0 (2020-01)
- CRI-O (2020-02)

コンテナレジストリに関しても、おそらくだいたい使える状態なのではないかと思います (docker.io、quay.ioでは使えます)。

[^5]: https://github.com/opencontainers/image-spec/pull/788

# zstd:chunkedとは

zstd:chunkedは、tar+zstdのメディアタイプの派生形です。

zstdを定義したRFC8878の[3.1 Frames](https://datatracker.ietf.org/doc/html/rfc8878#name-frames)に次のような記載があります: 

> Zstandard compressed data is made up of one or more frames. Each frame is independent and can be decompressed independently of other frames. The decompressed content of multiple concatenated frames is the concatenation of each frame's decompressed content.
> 
> There are two frame formats defined for Zstandard: Zstandard frames and skippable frames. Zstandard frames contain compressed data, while skippable frames contain custom user metadata.

つまりzstdには、「個々のファイルをzstdで圧縮したものを連結して、それを伸長すると、もとのファイル群を連結したものになる」という性質があります。
この性質を活用してコンテナレイヤーをファイル単位で扱えるようにしたのがzstd:chunkedです。

zstd:chunkedをサポートしないコンテナランタイムの場合でも、tar+zstdのメディアタイプをサポートしていれば、通常のコンテナイメージとして処理できます。その場合は、ファイル単位のダウンロードはできず、レイヤー単位での処理になります (がzstdの圧縮効率の良さという恩恵は受けることができます)。

zstd:chunkedは下記のバージョンから使えます。

- containers/storage v1.31.0 (2021-05)
- containers/image v5.14.0 (2021-07)
- Podman v3.3.0 (2021-08)
- CRI-O v1.22.0 (2021-08)

## 余談

gzipもzstdと同じく「個々にgzipしたものを連結しても全体をgzipストリームとして扱える」という性質を持ちます。Googleの[CRFS(Container Registry Filesystem)](https://github.com/google/crfs)というプロジェクトで、この性質を使ったStargz (Seekable tar.gz)というフォーマットが提案されました。zstd:chunkedはCRFSをもとに、zstd圧縮フォーマットを使って実装した機能です。同じ考え方でtar+gzのメディアタイプに対してファイル単位でのレイヤーの取り扱いおよびlazy pullingを実現したのがeStargzです。containerdのsnapshotter(コンテナで使用するファイルシステムのライフサイクルを管理するコンポーネント)のひとつに[stargz-snapshotter](https://github.com/containerd/stargz-snapshotter)があり、そこで使われているのがeStargzというイメージフォーマットです。

eStargzについては下記資料をご参照ください。
- [eStargzイメージとlazy pullingによる高速なコンテナ起動](https://www.slideshare.net/slideshow/estargzlazy-pulling/251120328)
- [Startup Containers in Lightning Speed with Lazy Image Distribution](https://www.slideshare.net/slideshow/startup-containers-in-lightning-speed-with-lazy-image-distribution/238069287)
- [Starting up Containers Super Fast With Lazy Pulling of Images](https://www.slideshare.net/slideshow/starting-up-containers-super-fast-with-lazy-pulling-of-images/244154126)
- [Speeding Up Pulling Container Images on a Variety of Tools with eStargz](https://medium.com/nttlabs/lazy-pulling-estargz-ef35812d73de)

eStargzはlazy pulling (ファイル単位で必要になったタイミングでpullする) が大きな特徴のひとつですが、containers/storageライブラリのzstd:chunked自体にはlazy pullingの機能はありません。

eStargzではtar+zstdも扱えます(zstdでのlazy pullingもできます)[^6]。
Podmanからstargz-snapshotterをAdditional Layer Storeとして利用することで、Podmanからlazy pullingを利用することもできます[^7][^8][^9]。

[^6]: https://github.com/containerd/stargz-snapshotter/pull/293
[^7]: https://github.com/containerd/stargz-snapshotter/pull/301
[^8]: https://github.com/containers/image/pull/1109
[^9]: https://github.com/containers/storage/pull/795

# zstd:chunkedの動き

zstd:chunkedのレイヤーフォーマットは下図のようになります (図は[c/storage PR#775](https://github.com/containers/storage/pull/775)から引用)。

![](https://user-images.githubusercontent.com/67430/102198255-73a98880-3ec2-11eb-9e05-93396e20ff6c.png)

zstd:chunkedでは、コンテナレイヤーの末尾にTOC (Table of Contents) 等のメタデータ情報が付与されています。レイヤーをpullするときは、

1. まずTOC情報をダウンロードする。TOCのオフセットはimage manifestのアノテーションに記載されている
2. TOCにリストされたレイヤー内のファイルごとに、
  a. 取得済みの全レイヤーのTOCをチェックし、もしそのファイルが手元にあれば、手元のファイルからハードリンク or reflink or コピーを作成する
  b. 手元になければ、HTTP Range Requestを使ってファイル単位でダウンロードする

という処理の流れになります。

メモリ消費の効率化を実現するためには、ハードリンクを使う必要があります。reflinkの場合、ファイルデータを共有するコピー元とコピー先のファイルのi-nodeが異なるため、カーネル(のVFSサブシステム)は同じファイルと認識できません。そのため、それらのファイルを読み込む際は、カーネルは別々のファイルとしてメモリにロードします。

一方で、ハードリンクを使うにあたっては注意が必要です。具体的には、ハードリンクを使った場合は

- ハードリンクを作成したファイル間で、i-nodeのメタデータ情報が同じになる (atime, mtime, ctime等)
- ファイルのリンクカウントが増える

という挙動になり、元のイメージのメタデータ情報を完全には再現できないためです。ファイルの同一性をSHA256のハッシュ値のみで判断しているのでしょうがないのですが...


# 動作確認

## 準備

zstd:chunkedの効果を確認するため、「ほとんど同じだけど一部だけファイルデータの異なるコンテナイメージ」を複数用意します。

具体的には、index.htmlだけが異なるApache httpdが入ったコンテナイメージを10個作ります。

```Dockerfile
FROM fedora:41
RUN dnf install -y httpd && dnf clean all
RUN echo "ServerName localhost" >> /etc/httpd/conf/httpd.conf && mkdir -p /var/www/html

ARG N
RUN echo "Container #${N}" > /var/www/html/index.html

EXPOSE 80
CMD ["httpd", "-D", "FOREGROUND"]
```

というDockerfileに対して

```sh
for i in $(seq 0 9); do
    podman build . -t quay.io/manabu.ori/myhttpd:c${i} --build-arg N=${i} --squash-all
    podman push --compression-format=zstd:chunked quay.io/manabu.ori/myhttpd:c${i}
done
```

で10個作成しました。zstd:chunkedの効果をわかりやすくするため、ビルド時に `--squash-all` をつけて、ベースイメージ含め全てを1個のレイヤーに押し込めています。
push時に `--compression-format=zstd:chunked` をつけると、zstd:chunkedフォーマットでイメージをpushします。


## reflink使用時の動き (xfsの場合)

### 設定

zstd:chunkedを使う際は、storage.confに下記を設定します。
`/usr/share/containers/storage.conf` を `/etc/containers/storage.conf` もしくは `~/.config/containers/storage.conf` にコピーして使ってください。

```ini
[storage.options]
pull_options = {enable_partial_images = "true", use_hard_links = "false", ostree_repos=""}
```

`enable_partial_images = "true"` でzstd:chunkedを有効にします。
以下では `use_hard_links = "false"` にし、reflinkを使う設定のときの動きを見てみます。

### 初期化

storage.confの設定をしたら、コンテナストレージを空にしておきます。

```
$ podman system reset
```

空になっていることがわかります。

```
$ find ~/.local/share/containers/storage/overlay -type f
/home/ori/.local/share/containers/storage/overlay/.has-mount-program
```

### 1個目のイメージ取得

1個目のイメージ(タグ `c0`)をpullします。

```
$ time podman pull quay.io/manabu.ori/myhttpd:c0
Trying to pull quay.io/manabu.ori/myhttpd:c0...
Getting image source signatures
Copying blob 4ca6d99209cf done  81.2MiB / 81.2MiB (skipped: 42.4KiB = 0.05%)
Copying config ddb744a7a7 done   |
Writing manifest to image destination
284abe58dc8218d85fa8c7d5b111149b8b57061fb2d89ca2130d44c70f6199bb

real	0m8.611s
user	0m2.785s
sys	0m1.015s
```

後で比較するため、filefragコマンドで/bin/lsのファイルデータの共有状況を確認しておきます (`flags` カラムに `shared` という表示がないため、現時点ではファイルデータを他のファイルと共有していません)。

```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 58465342
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       3:     123108..    123111:      4:
   1:        4..     143:   39138060..  39138199:    140:     123112: last,eof
```

### 2個目のイメージ取得

`c1` タグのレイヤーをpullすると、レイヤーの97.87%をスキップし、必要なチャンクのみダウンロードしたことがわかります (この例ではコンテナイメージ自体がそれほど大きくないため、pullにかかった時間自体はそれほど速くなったようには見えませんが...)。

```
$ time podman pull quay.io/manabu.ori/myhttpd:c1
Trying to pull quay.io/manabu.ori/myhttpd:c1...
Getting image source signatures
Copying blob 63df9f9dafda done  81.2MiB / 81.2MiB (skipped: 79.4MiB = 97.87%)
Copying config 0ca3f4f445 done   |
Writing manifest to image destination
9abebd0ec4a754fb65e2ab985c89b71b9fa339a23606e0288d6526e78a4d47dd

real	0m6.728s
user	0m0.719s
sys	0m1.109s
```

reflinkを作っているため、ファイルデータは同じファイルでも、i-node番号が異なります。

```
$ ls -i ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
241127 /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
$ ls -i ~/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
7309808 /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
```

filefragの出力を見ると、`flags` カラムに `shared` が含まれることから、ファイルデータが他のファイルと共有されていることがわかります。

```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 58465342
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       3:     123108..    123111:      4:             shared
   1:        4..     143:   39138060..  39138199:    140:     123112: last,shared,eof
/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls: 2 extents found
```

reflinkの場合、`du` コマンドで見ても、データを共有しているかはわからないので注意が必要です。

## reflink使用時の動き (btrfsの場合)

同じように、btrfsを使った場合にタグ `c0` と `c1` の2つをpullします。filefragコマンドの `flags` カラムの見え方がxfsのときと少し異なりますが、`shared` があるのでreflinkでデータを共有できていることがわかります。

```
$ filefrag -ek ~/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
Filesystem type is: 9123683e
File size of /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls is 145480 (144 blocks of 1024 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..     127:    2438128..   2438255:    128:             encoded,shared
   1:      128..     143:    2424244..   2424259:     16:    2438256: last,encoded,shared,eof
/home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls: 2 extents found
```

btrfsの場合は、`btrfs filesystem du` コマンドで、reflinkによるファイルデータの共有具合を確認することができます (`Set shared` カラム)。

```
$ sudo btrfs filesystem du -s ~/.local/share/containers/storage/overlay/*
     Total   Exclusive  Set shared  Filename
 218.59MiB    11.46MiB   120.48MiB  /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77
 209.22MiB       0.00B   120.48MiB  /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/l
     0.00B       0.00B       0.00B  /home/ori/.local/share/containers/storage/overlay/staging
```

## ハードリンク使用時の動き

### 設定

storage.confの `pull_options` に `use_hard_links = "true"` を設定し、コンテナストレージを空にします。

```
$ crudini --get /etc/containers/storage.conf storage.options pull_options
{enable_partial_images = "true", use_hard_links = "true", ostree_repos=""}
```

```
$ podman system reset
```

### ストレージ使用量の確認

コンテナイメージを10個pullします。

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

各イメージのディレクトリで `du` を実行すると、2番目以降にpullしたイメージのディレクトリの使用量が小さいことがわかります。

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

ハードリンクを作成しているため、同じファイルのi-node番号は等しくなります。

```
$ ls -i ~/.local/share/containers/storage/overlay/*/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/00db55db807cab800fdf6b6ed39ec46e8ec53c1e4b755893f9f6d30d500fe4d0/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/09a82655f255757ec1d03b8d81b9cd92f0ffcf987d31c2b7ef1f0f5205777d77/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/30fec452bc3836be305803c342af9f11a5ca6e16f38496176b6c80bee168ae69/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/36fc922787f2a9132ba3d7ea9decc8db6203518c6e05a61a049b91f6c3c66f81/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/480110807cafc058472a3f2eea4d8f9a57bc0a528d6ae21108bf05475e4d28c7/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/79241936613eb087b6b84a3939bbbadcb0bb198b089c65c90e916f9b06688123/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/c66d4f5eccdb07b29d460fa0c23af76e222a9ae97aca9134ac55370e4c466130/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/f1674671929ad75874ecd2bf1fc394ee394a1802acafcb899c7a703c194c7cb7/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/f401afca4b05f6a3dfa89dd0caf6f7cbc5c3a8706d1f42084ddabac370b7ba88/diff/usr/bin/ls
3221517073 /home/ori/.local/share/containers/storage/overlay/f72b3dfebd3aca217b7f0d5ef0a75e50567a696faafd058b1176cd00ef80f418/diff/usr/bin/ls
```

### メモリ使用量の確認

コンテナを10個起動します (結果をわかりやすくするため、entrypointをsleepコマンドにしました。httpdはforkしたりするので...)。

```
$ for i in $(seq 0 9); do ((port = i + 9000)); podman run --name myhttpd-c${i} --rm -d -p ${port}:80 quay.io/manabu.ori/myhttpd:c${i} sleep infinity; done
```

[smem](https://www.selenic.com/smem/)コマンドで、ハードリンク使用時のsleepのメモリ使用量を表示すると下記のようになります。`USS` がUnique Set Size、つまりシェアされていないプロセス固有のメモリ領域のサイズを表します。

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

ちなみにreflink使用時の場合は下記のようになります。

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

ハードリンク使用時はページキャッシュの共有が効いており、メモリ使用量が抑えられていることがわかります。

# 参考文献
- Red Hat Blog: [Pull container images faster with partial pulls](https://www.redhat.com/en/blog/faster-container-image-pulls)
- [Enable zstd:chunked support in containers/image #775](https://github.com/containers/storage/pull/775)