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

コンテナの文脈では、composefsは「ファイル単位で重複排除ができるコンテナストレージ」を提供するファイルシステムと言えます。通常のコンテナイメージは、レイヤーごとにSHA256のハッシュ値を計算し、複数のコンテナイメージで同じレイヤーを使用する場合はそのレイヤーを共有することで、イメージレイヤーごとの重複排除を行います。composefsを使用すると、より粒度の細かいファイル単位での重複排除行うことができます (複数のレイヤーに同じバイナリが入っている場合、1個分のストレージ容量しか使いません)。

なんでOpenShiftとは関係ないcomposefsをAdvent Calendarのネタにしているかといいますと... OpenShiftのノードは「RHEL CoreOS」というLinuxディストリビューションを使っているのですが、そのアップストリームであるFedora CoreOSでは、実はcomposefsが使われています。というわけで、それほど遠くない将来、OpenShiftに入ってくる(かもしれない)機能[^1]ということで、ご容赦ください... 

[^1]: 外から見えるJIRAチケットもありました https://issues.redhat.com/browse/COS-2963

# composefsの機能

composefsは、いくつかの特徴を持つ新しいファイルシステムです。

典型的なファイルシステムはブロックデバイスをバックエンドに持ちますが、対してcomposefsは通常のread-onlyのファイルの集合をバックエンドに持ちます。別の言い方をすると、「バックエンドとなる通常のファイルの集合 (＋メタデータの情報) に対してcomposefsマウントする」という、ある種のループバックマウントをするファイルシステムです。

バックエンドとなるファイル群は、「content-addressed backing files」と呼ばれます。content addressedという言葉は、ファイルの内容からファイルを指定できる、という気持ちが込められています。具体的には、バックエンドのファイルは、ファイルの中身のSHA256のハッシュ値がファイル名となります。ネットワーク機器には搭載されているCAM (Content Addressable Memory, 連想メモリ) が想起される言葉です。MACテーブルやルーティングテーブルの検索をハードウェアで高速処理する。

composefsは、fs-verityを使ってファイルシステム全体の改ざん検知ができます。fs-verityはLinuxカーネルが持つファイル単位の改ざん検知を行う仕組みです。fs-verity自体は、ファイルの中身にしか関与しません。つまりfs-verytyではメタデータの変更は検知できません。しかしcomposefsの場合は、ファイルシステムのメタデータ/ディレクトリツリーをEROFSのイメージとして持つため、このイメージファイルに対してもfs-verityのダイジェストを確認することで、ファイルシステム全体の改ざん検知ができます。

# 使い方

composefsで遊ぶためのディレクトリツリーを適当に作ります。

```
$ tree rootfs
rootfs
├── foo.txt
├── subdir
│   └── bar.txt
└── testfile

2 directories, 3 files
```



```
$ python -c 'print("foo.txt" + "_" * 60)' > rootfs/foo.txt
$ python -c 'print("bar.txt" + "_" * 60)' > rootfs/subdir/bar.txt
$ echo 'abcde' > rootfs/testfile
```


```
$ mkcomposefs --digest-store=objects rootfs example.cfs
```

example.cfsというerofsのイメージファイルと、objectsというディレクトリ配下にいくつかのファイルが作成されます。

```
$ file example.cfs
example.cfs: EROFS filesystem, compat: MTIME, blocksize=12, exslots=0, uuid=00000000-0000-0000-0000-000000000000
```

```
$ tree objects
objects
├── 85
│   └── d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a
└── fc
    └── 2a1a56808b1739e0fb1621d2170b42d9cfd57c54f7481b1c29935e440fd8a4

3 directories, 2 files
```

マウントポイントを作ってマウントします。

```
$ sudo mkdir -p /mnt/composefs
$ sudo mount -t composefs -o basedir=objects example.cfs /mnt/composefs
```

```
$ findmnt -J | jq '.filesystems[].children[] | select(.target == "/mnt/composefs")'
{
  "target": "/mnt/composefs",
  "source": "composefs",
  "fstype": "overlay",
  "options": "ro,relatime,seclabel,lowerdir+=/tmp/.composefs.aONLgi,datadir+=objects,redirect_dir=on,metacopy=on"
}
```

mkcomposefsで生成されたerofsのイメージファイルがloopbackマウントされていることがわかります。

```
$ losetup
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                            DIO LOG-SEC
/dev/loop0         0      0         1  1 /home/ori/work/composefs/example.cfs   1    4096
```

/mnt/composefsにはrootfsと同じ内容のファイルが見えます。

```
$ tree /mnt/composefs
/mnt/composefs
├── foo.txt
├── subdir
│   └── bar.txt
└── testfile

2 directories, 3 files
```

```
$ cat /mnt/composefs/foo.txt
foo.txt____________________________________________________________
$ cat /mnt/composefs/subdir/bar.txt
bar.txt____________________________________________________________
$ cat /mnt/composefs/testfile
abcde
```

erofsのイメージファイルもマウントしてみましょう。

```
$ sudo mkdir -p /mnt/erofs
$ sudo mount -oloop /dev/loop0 /mnt/erofs
```

```
$ findmnt -J | jq '.filesystems[].children[] | select(.target == "/mnt/erofs")'
{
  "target": "/mnt/erofs",
  "source": "/dev/loop1",
  "fstype": "erofs",
  "options": "ro,relatime,seclabel,user_xattr,acl,cache_strategy=readaround"
}
```

```
$ ls -l /mnt/erofs
total 12
crw-r--r--. 1 ori ori 0, 0 Dec  4 15:06 00
crw-r--r--. 1 ori ori 0, 0 Dec  4 15:06 01
...
crw-r--r--. 1 ori ori 0, 0 Dec  4 15:06 fe
crw-r--r--. 1 ori ori 0, 0 Dec  4 15:06 ff
-rw-r--r--. 1 ori ori   68 Dec  4 15:07 foo.txt
drwxr-xr-x. 2 ori ori   46 Dec  4 11:45 subdir
-rw-r--r--. 1 ori ori    6 Dec  4 15:07 testfile
```

maj 0, min 0のchardevファイルがたくさんと、あとrootfsにあったファイルが見えます。それぞれファイルの拡張属性を見てみましょう。

```
$ sudo getfattr -d -m - /mnt/erofs/foo.txt
getfattr: Removing leading '/' from absolute path names
# file: mnt/erofs/foo.txt
security.selinux="unconfined_u:object_r:user_home_t:s0"
trusted.overlay.metacopy=0sACQAAYXWANRi9cNzi1XD6/VwwxJjNT3GqjVEjGqPmqUZQpyK
trusted.overlay.redirect="/85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a"
```

```
$ sudo getfattr -d -m - /mnt/erofs/subdir/bar.txt
getfattr: Removing leading '/' from absolute path names
# file: mnt/erofs/subdir/bar.txt
security.selinux="unconfined_u:object_r:user_home_t:s0"
trusted.overlay.metacopy=0sACQAAfwqGlaAixc54PsWIdIXC0LZz9V8VPdIGxwpk15ED9ik
trusted.overlay.redirect="/fc/2a1a56808b1739e0fb1621d2170b42d9cfd57c54f7481b1c29935e440fd8a4"
```

```
$ sudo getfattr -d -m - /mnt/erofs/testfile
getfattr: Removing leading '/' from absolute path names
# file: mnt/erofs/testfile
security.selinux="unconfined_u:object_r:user_home_t:s0"
```

`foo.txt`、`subdir/bar.txt` には `trusted.overlay.redirect` の拡張属性が設定されており、mkcomposefs実行時に `--digest-store` で指定したオブジェクトストアに生成されたファイルのパスが書かれています。

composefsでファイルにアクセスすると、まずerofsのディレクトリツリーをたどって指定したパスを探し、そのファイルの拡張属性 `trusted.overlay.redirect` を見てオブジェクトストア内のファイルに到達します。foo.txtの場合は、オブジェクトストア内の `85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a` にそのデータが入っています。

```
$ cat /mnt/composefs/foo.txt
foo.txt____________________________________________________________
$ sudo getfattr -d -m - /mnt/erofs/foo.txt
getfattr: Removing leading '/' from absolute path names
# file: mnt/erofs/foo.txt
security.selinux="unconfined_u:object_r:user_home_t:s0"
trusted.overlay.metacopy=0sACQAAYXWANRi9cNzi1XD6/VwwxJjNT3GqjVEjGqPmqUZQpyK
trusted.overlay.redirect="/85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a"
$ cat objects/85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a
foo.txt____________________________________________________________
```

ちなみに、ファイルサイズが64バイト未満の場合は、オブジェクトストアにリダイレクトせず、erofsのイメージ内に直接ファイルデータを埋め込みます。上記の例で、オブジェクトストア内にファイルが2個しかなく、erofsイメージ内のtestdataの拡張属性 `trusted.overlay.redirect` が設定されていなかったのはそのためです。

まとめると、composefsを作ると

- 元になるディレクトリ以下のファイルは、その中身のハッシュ値をファイル名としたファイルとして、オブジェクトストア内にコピーされる (細かいことを言うと、reflinkが使えるファイルシステムの場合はreflinkが作られます)
- 元のディレクトリツリーおよびメタデータの情報は、mkcomposefsで生成されるerofsイメージに格納されます。このとき、各ファイルの拡張属性に、対応するオブジェクトストア内のファイルのパスが入ります。
- ファイルサイズが小さい場合は、オブジェクトストアにリダイレクトせず、erofsイメージにデータを直接埋め込みます

のようになります。

# Podmanからcomposefsを使う

Podmanでcomposefsを使う場合、現時点ではrootlessはサポートされません[^2]。rootfulで実行する必要があります。

[^2]: たぶん問題はerofsのイメージをloopbackマウントしているところだと思われます。loopbackマウントってnamespace対応していないとか、root権限が必要とか、いろいろとrootlessに厳しい状況なので...

## 準備

storage.confの `[storage.options.overlay]` セクションで `use_composefs = "true"` を設定します。

- 設定値は `"true"` (文字列) です
- 現時点では、storage.confに関しては `/etc/containers/storage.conf.d` を使った設定はサポートされていないので[^3]、`/usr/share/containers/storage.conf` を編集する必要があります

の2点に注意してください。

[^3]: PRは出ています https://github.com/containers/storage/pull/1885

storage.confを編集したら、`sudo podman system reset` を実行しておきます。

## コンテナイメージ

Podmanでcomposefsを使用するためには、MIMETypeが `application/vnd.oci.image.layer.v1.tar+zstd` のコンテナイメージを使う必要があります。おそらく世の中にあるベースイメージで使うコンテナイメージでzstdになっているものはほとんどないと思われますが、今のPodmanはデフォルトでzstd:chunkedを使う設定になっていますので、適当に手元でDockerfileを書いてbuild & pushすると、zstdのコンテナイメージとしてレジストリにpushします。以下の例では `quay.io/manabu.ori/myhttpd` を使用します。このイメージはzstdで圧縮したものになっています。

```
# skopeo inspect docker://quay.io/manabu.ori/myhttpd | jq '.LayersData[]'
{
  "MIMEType": "application/vnd.oci.image.layer.v1.tar+zstd",
  "Digest": "sha256:a6faeb3e4b76c470d82c18b84dc4b4a015cc0b89e8cef9bd763657b667b6174c",
  "Size": 86901108,
  "Annotations": {
    "io.github.containers.zstd-chunked.manifest-checksum": "sha256:d605c93102081795272182d023c092e168a0b8b35532a17e298bbc9161de2b4c",
    "io.github.containers.zstd-chunked.manifest-position": "85604264:836101:4162774:1",
    "io.github.containers.zstd-chunked.tarsplit-position": "86440373:460663:11131307"
  }
}
...
```

## コンテナの実行

```
podman run --name myhttpd --rm -d -p 8080:80 quay.io/manabu.ori/myhttpd
```

loopbackマウントの様子を確認します。

```
# losetup
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                                                                                          DIO LOG-SEC
/dev/loop1         0      0         1  1 /var/lib/containers/storage/overlay/157d2a4e319e4f4d6373e76665d4e996b96d06086acc09e415c792dde6b27175/composefs-data/composefs.blob   0     512
/dev/loop2         0      0         1  1 /var/lib/containers/storage/overlay/e175d5fd713e8bf1d99f9a8d2d9f453254b22be9719c39ce13d1c1355d6586e1/composefs-data/composefs.blob   0     512
/dev/loop0         0      0         1  1 /var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/composefs-data/composefs.blob   0     512
/dev/loop3         0      0         1  1 /var/lib/containers/storage/overlay/00753547945b10ec7a54f62b5e1b59bb440692f608554defc4245efcd46d7987/composefs-data/composefs.blob   0     512
```

コンテナストレージのoverlayfsマウントの様子を確認します。

```
# findmnt -J | jq '.filesystems[].children[] | select(.target == "/var/lib/containers/storage/overlay")'
{
  "target": "/var/lib/containers/storage/overlay",
  "source": "/dev/nvme0n1p3[/root/var/lib/containers/storage/overlay]",
  "fstype": "btrfs",
  "options": "rw,relatime,seclabel,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=256,subvol=/root",
  "children": [
    {
      "target": "/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/merged",
      "source": "overlay",
      "fstype": "overlay",
      "options": "rw,nodev,relatime,context=\"system_u:object_r:container_file_t:s0:c699,c861\",lowerdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/1:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/2:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/3:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/4::/var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/diff::/var/lib/containers/storage/overlay/157d2a4e319e4f4d6373e76665d4e996b96d06086acc09e415c792dde6b27175/diff::/var/lib/containers/storage/overlay/e175d5fd713e8bf1d99f9a8d2d9f453254b22be9719c39ce13d1c1355d6586e1/diff::/var/lib/containers/storage/overlay/00753547945b10ec7a54f62b5e1b59bb440692f608554defc4245efcd46d7987/diff,upperdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/diff,workdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/work,redirect_dir=on,uuid=on,metacopy=on,volatile"
    }
  ]
}
```

実行中のコンテナのoverlayfsマウントのオプションを紐解いてみます。

```
rw,
nodev,
relatime,
context="system_u:object_r:container_file_t:s0:c699,c861",
lowerdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/1:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/2:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/3:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/4::/var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/diff::/var/lib/containers/storage/overlay/157d2a4e319e4f4d6373e76665d4e996b96d06086acc09e415c792dde6b27175/diff::/var/lib/containers/storage/overlay/e175d5fd713e8bf1d99f9a8d2d9f453254b22be9719c39ce13d1c1355d6586e1/diff::/var/lib/containers/storage/overlay/00753547945b10ec7a54f62b5e1b59bb440692f608554defc4245efcd46d7987/diff,
upperdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/diff,
workdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/work,
redirect_dir=on,
uuid=on,
metacopy=on,
volatile
```

さらにlowerdirを見やすく整形すると...

```
lowerdir=/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/1
:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/2
:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/3
:/var/lib/containers/storage/overlay/335b77ce2a77a0fcd238625f641490ecb6c10a5ed0c52e4e659df85438193389/composefs-layers/4
::/var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/diff
::/var/lib/containers/storage/overlay/157d2a4e319e4f4d6373e76665d4e996b96d06086acc09e415c792dde6b27175/diff
::/var/lib/containers/storage/overlay/e175d5fd713e8bf1d99f9a8d2d9f453254b22be9719c39ce13d1c1355d6586e1/diff
::/var/lib/containers/storage/overlay/00753547945b10ec7a54f62b5e1b59bb440692f608554defc4245efcd46d7987/diff,
```

コロンが2個連続しているレイヤーが data only lower layers で、この中のファイルはcomposefsにおけるオブジェクトストア (content addressed backing filesの置き場) であることを表しています。そのうちの一つを覗いてみましょう。

```
# mount -oloop /var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/composefs-data/composefs.blob /mnt/tmp
# cat /mnt/tmp/var/www/html/index.html
# getfattr -d -m - /mnt/tmp/var/www/html/index.html
getfattr: Removing leading '/' from absolute path names
# file: mnt/tmp/var/www/html/index.html
trusted.overlay.metacopy=0sACQAAeG6n0gwFNmWFkJWaUL9QlUwHYFLULh0Z7qRFqFgm3iQ
trusted.overlay.redirect="/f1/f2463d6c783031a0b34d4c545b4383a1d24a3c6efefc33cc0cd6ca339b0dea"
```

```
# cat /var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/diff/f1/f2463d6c783031a0b34d4c545b4383a1d24a3c6efefc33cc0cd6ca339b0dea
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Fedora Container</title>
</head>
<body>
    <h1>Welcome to Fedora Container!</h1>
</body>
</html>
```

# 歴史

最初は、新規の独自ファイルシステムという形で、2022年11月に最初のRFC patchがLKMLに投稿されました。この後議論を重ねながらv2, v3とパッチが更新されますが、どちらかというとあまりカーネルコミュニティからの賛同は得られませんでした。むしろ提案されたユースケースであれば、新しいファイルシステムを作るよりも、似た機能を持つ既存の仕組み (overlayfs、erofs等) を改良する方がよいのではないか、という意見が出ました。

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

# footnotes

[

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

