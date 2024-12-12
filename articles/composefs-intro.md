---
title: "コンテナストレージに特化したファイルシステムComposeFSとは"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに

本記事は、OpenShift Advent Calendar 2024の12/11のエントリーで、[composefs](https://github.com/containers/composefs)というコンテナ環境向けのファイルシステムを紹介します。

composefsは、現在のところ、コンテナストレージおよびOSTreeベースのOSイメージのリポジトリとしてのユースケースが考えられています。「OSTreeベースのOSイメージのリポジトリ」とは、[bootc](https://github.com/containers/bootc)を使ってOSのファイルシステムをコンテナイメージとして管理するときのOSTreeリポジトリ置き場として使うことを意味します (bootcについては以前、下のブログ記事を書きました)。

https://zenn.dev/orimanabu/articles/try-rhel-image-mode

先日のKubeCon NAで、[Podmanをはじめとするコンテナ関連ツール群をCNCFに寄贈する旨の発表](https://www.redhat.com/en/blog/red-hat-contribute-comprehensive-container-tools-collection-cloud-native-computing-foundation)がありましたが、その中にcomposefsとbootcも含まれています。

なんでOpenShiftとは関係ないcomposefsをAdvent Calendarのネタにしているかといいますと... OpenShiftのノードは「RHEL CoreOS」というLinuxディストリビューションを使っているのですが、そのアップストリームであるFedora CoreOSでは、実はcomposefsが使われています。というわけで、それほど遠くない将来、OpenShiftに入ってくる(かもしれない)機能[^1]ということで、ご容赦ください... 

[^1]: 外から見えるJIRAチケットありました https://issues.redhat.com/browse/COS-2963

# composefsの機能

composefsは、いくつかの特徴を持つ新しいファイルシステムです。

GitHubのREADMEには

> "The reliability of disk images, the flexibility of files"

とあります。ディスクイメージはtar/zipと比べて、マウントできる、dm-verityを使って改ざんを検知できる等、カーネルと連携しやすいという特徴があります。一方で、必要以上のストレージ容量が必要だったり、取り回しの柔軟性に欠ける傾向があります。composefsは、ファイルベースの実装で柔軟さを維持しつつ、改ざん検知やカーネルによるマウントをサポートするファイルシステムです。

典型的なファイルシステムはブロックデバイスをマウントしますが、composefsはext4やxfsといった通常のファイルシステム上のファイルの集合をバックエンドに持ちます。別の言い方をすると、「バックエンドとなる通常のファイルの集合 (＋ファイルツリー/メタデータの情報) に対してcomposefsマウントする」という、ある種のループバックマウントをするファイルシステムという言い方もできるかもしれません。

バックエンドとなるファイル群の置き場は、「content-addressed object store」(もしくは単に「オブジェクトストア」)と呼ばれます。content addressedという言葉は、ファイルの内容からファイルを指定できる、という気持ちが込められています。具体的には、バックエンドのファイルは、ファイルの中身のSHA256のハッシュ値がファイル名となります。ネットワーク機器には搭載されているCAM (Content Addressable Memory, MACテーブルやルーティングテーブルの検索をハードウェアで高速処理するやつ) が想起される言葉ですね。

composefsは、erofs、overlayfs、fs-verityといったカーネルの機能を組み合わせて使います。具体的には、

- 通常のext4やxfsといったファイルシステム上にオブジェクトストアを作り、そこにSHA256のハッシュ値をファイル名とするファイルコンテンツを配置する
- ファイルシステムのファイルツリーやメタデータは、erofsのイメージファイルに入れる
- erofsのイメージファイルとオブジェクトストアをoverlayfsで重ね合わせてマウントする

という動きになります。ファイルのメタデータとデータを別のファイルシステムに持ち、overlayfsで組み合わせる、という機能は、composefsのために開発されました。この辺りの細かい話は後述します。

composefsはfs-verityを使ってファイルシステム全体の改ざん検知ができます。fs-verityはLinuxカーネルが持つファイル単位の改ざん検知を行う仕組みです。fs-verity自体はファイルのデータ(中身)にしか関与しません。つまりfs-verytyではメタデータの変更は検知できません。しかしcomposefsの場合は、ファイルシステムのメタデータ/ディレクトリツリーをEROFSのイメージとして持つため、このイメージファイルに対してもfs-verityのダイジェストを確認することで、メタデータを含めたファイルシステム全体の改ざん検知ができます。

ファイルデータのオブジェクトストアと、ファイルツリー/メタデータのイメージを別に持つことで、

- 複数のファイルシステムで同じ中身を持つファイルがある場合は、ファイルデータをcontent-addressedなオブジェクトストアに1個持つだけで済む (ファイル単位の重複排除)
- ひとつのオブジェクトファイルを、異なるメタデータを持つファイルとして複数のファイルシステムに見せることができる

という利点があります。

余談ですが、SteamOSのOSアップデートの仕組みで使っている[RAUC](https://github.com/rauc/rauc/pull/1500)というOSSプロジェクトでも、composefsを使うようになるようです[^2]。

https://cfp.all-systems-go.io/all-systems-go-2024/talk/3DKX9V/

[^2] https://github.com/rauc/rauc/pull/1500

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

ファイルの中身を適当に詰めます。

```
$ python -c 'print("foo.txt" + "_" * 60)' > rootfs/foo.txt
$ python -c 'print("bar.txt" + "_" * 60)' > rootfs/subdir/bar.txt
$ echo 'abcde' > rootfs/testfile
```

mkcomposefsコマンドを実行します。`--digest-store` オプションでオブジェクトストアを指定し、コマンドライン引数でcomposefsの元となるディレクトリと、メタデータを入れるイメージファイル名を指定します。

```
$ mkcomposefs --digest-store=objects rootfs example.cfs
```

指定したexample.cfsというerofsのイメージファイルと、objectsというディレクトリ配下にいくつかのファイルが作成されます。

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

マウントポイントを作ってcomposefsでマウントします。`-o basedir` でオブジェクトストアを指定します。

```
$ sudo mkdir -p /mnt/composefs
$ sudo mount -t composefs -o basedir=objects example.cfs /mnt/composefs
```

無事 `/mnt/composefs` にcomposefsでマウントできました。

```
$ findmnt -J | jq '.filesystems[].children[] | select(.target == "/mnt/composefs")'
{
  "target": "/mnt/composefs",
  "source": "composefs",
  "fstype": "overlay",
  "options": "ro,relatime,seclabel,lowerdir+=/tmp/.composefs.aONLgi,datadir+=objects,redirect_dir=on,metacopy=on"
}
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

mkcomposefsで生成されたerofsのイメージファイルを使って、ループバックデバイスがこっそり作成されています。
(「erofsでマウントする」という情報はmount.composefsが隠蔽しており、findmntで見てもerofsの情報は出てきません)

```
$ losetup
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                            DIO LOG-SEC
/dev/loop0         0      0         1  1 /home/ori/work/composefs/example.cfs   1    4096
```

erofsのイメージファイルもマウントしてみましょう。
下の例ではループバックデバイスをマウントしていますが、erofsfuseコマンドで、erofsイメージファイルを直接fuseマウントすることもできます。

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

想定ユースケースその1、コンテナストレージです。

Podmanでcomposefsを使う場合、現時点ではrootlessはサポートされません[^3]。rootfulで実行する必要があります。

[^3]: たぶん問題はerofsのイメージをループバックマウントしているところだと思われます。ループバックマウントってnamespace対応していないとか、root権限が必要とか、いろいろとrootlessに厳しい状況なので...

## 準備

storage.confの `[storage.options.overlay]` セクションで `use_composefs = "true"` を設定します。

- 設定値は `"true"` (文字列) です
- 現時点では、storage.confに関しては `/etc/containers/storage.conf.d` を使った設定はサポートされていないので[^4]、`/usr/share/containers/storage.conf` を編集する必要があります

の2点に注意してください。

[^4]: PRは出ています https://github.com/containers/storage/pull/1885

storage.confを編集したら、`sudo podman system reset` を実行しておきます。

## コンテナイメージ

Podmanでcomposefsを使用するためには、メディアタイプが `application/vnd.oci.image.layer.v1.tar+zstd` のコンテナイメージを使う必要があります。おそらく世の中にあるベースイメージで使うコンテナイメージでzstdになっているものはほとんどないと思われますが、今のPodmanはデフォルトでzstd:chunkedを使う設定になっていますので、適当に手元でDockerfileを書いてbuild & pushすると、zstdのコンテナイメージとしてレジストリにpushします。以下の例では `quay.io/manabu.ori/myhttpd` を使用します。このイメージはzstdで圧縮したものになっています。

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

ループバックマウントの様子を確認します。

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

# イメージモードのOS領域で使う

想定ユースケースその2、OSTreeのバックエンドで使うパターンです。

これは少し背景の説明が必要かもしれません。[OSTree](https://ostreedev.github.io/ostree/)は、ファイルシステムをgitのように管理し、ファイルシステムツリーにSHA256 IDをつけ、ファイルシステム全体をアトミックに切り替えられるようにする仕組みです。

## bootc

```
podman-bootc run --filesystem=xfs quay.io/manabu.ori/fedora-bootc-test:41
```

```
# cat /etc/os-release
NAME="Fedora Linux"
VERSION="41.20241210.0 (Forty One)"
RELEASE_TYPE=stable
ID=fedora
VERSION_ID=41
VERSION_CODENAME=""
PLATFORM_ID="platform:f41"
PRETTY_NAME="Fedora Linux 41.20241210.0 (Forty One)"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:41"
DEFAULT_HOSTNAME="fedora"
HOME_URL="https://fedoraproject.org/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora/f41/system-administrators-guide/"
SUPPORT_URL="https://ask.fedoraproject.org/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=41
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=41
SUPPORT_END=2025-12-15
OSTREE_VERSION='41.20241210.0'
```

```
# cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/boot/ostree/default-fb1fef48e415876a4a43c311b1ef3c5ecfea7094e0befb2518c631fca55deeb4/vmlinuz-6.11.10-300.fc41.x86_64 root=UUID=da43332a-c607-45a9-808e-1eaa0ec68ba5 rw ostree=/ostree/boot.1/default/fb1fef48e415876a4a43c311b1ef3c5ecfea7094e0befb2518c631fca55deeb4/0
```

```
# strings /run/ostree-booted
backing-root-device-inode
(tt)
composefs
root.transient
sysroot-ro
6Ld|
```

```
# systemctl | grep -E '(coreos|bootc)-'
● bootc-fetch-apply-updates.service                                            loaded failed failed    Apply bootc updates
  bootc-fetch-apply-updates.timer                                              loaded active waiting   Apply bootc updates
```

```
# findmnt -J /
{
   "filesystems": [
      {
         "target": "/",
         "source": "composefs",
         "fstype": "overlay",
         "options": "ro,relatime,seclabel,lowerdir+=/run/ostree/.private/cfsroot-lower,datadir+=/sysroot/ostree/repo/objects,redirect_dir=on,metacopy=on"
      }
   ]
}
```

```
# losetup
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                                                                            DIO LOG-SEC
/dev/loop0         0      0         1  1 /sysroot/ostree/deploy/default/deploy/85dda4cd46c001a4e2c21a3b835a2d1ca9d69a5b6687e5d86b913083783a5594.0/.ostree.cfs   1    4096
```

```
# mount -oloop /sysroot/ostree/deploy/default/deploy/85dda4cd46c001a4e2c21a3b835a2d1ca9d69a5b6687e5d86b913083783a5594.0/.ostree.cfs /mnt
mount: /var/mnt: WARNING: source write-protected, mounted read-only.
```

```
# getfattr -d -m - /mnt/usr/share/containers/storage.conf
getfattr: Removing leading '/' from absolute path names
# file: mnt/usr/share/containers/storage.conf
security.selinux="system_u:object_r:usr_t:s0"
trusted.overlay.metacopy=""
trusted.overlay.redirect="/65/d4629a8032125b9a43b66d9c1633af34ab953ab1dd30dadfac769c925b03bb.file"
```

```
# head -n 10 /sysroot/ostree/repo/objects/65/d4629a8032125b9a43b66d9c1633af34ab953ab1dd30dadfac769c925b03bb.file
# This file is the configuration file for all tools
# that use the containers/storage library. The storage.conf file
# overrides all other storage.conf files. Container engines using the
# container/storage library do not inherit fields from other storage.conf
# files.
#
#  Note: The storage.conf file overrides other storage.conf files based on this precedence:
#      /usr/containers/storage.conf
#      /etc/containers/storage.conf
#      $HOME/.config/containers/storage.conf
```

```
# rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/manabu.ori/fedora-bootc-test:41
                   Digest: sha256:cee29d9d76c2fdb1574190878a2c99e1b3fc5bf226680be366bbdcfced874986
                  Version: 41.20241210.0 (2024-12-10T14:44:30Z)
```

```
# bootc status
No staged image present
Current booted image: quay.io/manabu.ori/fedora-bootc-test:41
    Image version: 41.20241210.0 (2024-12-10 14:44:30.660720065 UTC)
    Image digest: sha256:cee29d9d76c2fdb1574190878a2c99e1b3fc5bf226680be366bbdcfced874986
No rollback image present
```


## Fedora CoreOS

```
$ cat /etc/os-release
NAME="Fedora Linux"
VERSION="41.20241109.3.0 (CoreOS)"
RELEASE_TYPE=stable
ID=fedora
VERSION_ID=41
VERSION_CODENAME=""
PLATFORM_ID="platform:f41"
PRETTY_NAME="Fedora CoreOS 41.20241109.3.0"
ANSI_COLOR="0;38;2;60;110;180"
LOGO=fedora-logo-icon
CPE_NAME="cpe:/o:fedoraproject:fedora:41"
HOME_URL="https://getfedora.org/coreos/"
DOCUMENTATION_URL="https://docs.fedoraproject.org/en-US/fedora-coreos/"
SUPPORT_URL="https://github.com/coreos/fedora-coreos-tracker/"
BUG_REPORT_URL="https://github.com/coreos/fedora-coreos-tracker/"
REDHAT_BUGZILLA_PRODUCT="Fedora"
REDHAT_BUGZILLA_PRODUCT_VERSION=41
REDHAT_SUPPORT_PRODUCT="Fedora"
REDHAT_SUPPORT_PRODUCT_VERSION=41
SUPPORT_END=2025-12-15
VARIANT="CoreOS"
VARIANT_ID=coreos
OSTREE_VERSION='41.20241109.3.0'
```

```
$ cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/boot/ostree/fedora-coreos-717f957fb4680546cce36bc1c5e633abdbf5e4ecb6e99e5665df3c00a56088fa/vmlinuz-6.11.6-300.fc41.x86_64 rw mitigations=auto,nosmt ostree=/ostree/boot.0/fedora-coreos/717f957fb4680546cce36bc1c5e633abdbf5e4ecb6e99e5665df3c00a56088fa/0 ignition.platform.id=qemu console=tty0 console=ttyS0,115200n8 root=UUID=4828f807-1e7f-419a-88f2-31df1a736fa1 rw rootflags=prjquota boot=UUID=a6192326-87ec-42a5-9653-ae5424995744
```

```
$ strings /run/ostree-booted
backing-root-device-inode
(tt)
composefs
root.transient
sysroot-ro
6Ld|
```

```
$ systemctl | grep -E '(coreos|bootc)-'
  sysroot-ostree-deploy-fedora\x2dcoreos-var.mount                                          loaded active mounted   sysroot-ostree-deploy-fedora\x2dcoreos-var.mount
  coreos-check-wireless-firmwares.service                                                   loaded active exited    Check For Wireless Firmware Packages
  coreos-ignition-write-issues.service                                                      loaded active exited    Create Ignition Status Issue Files
  coreos-platform-chrony-config.service                                                     loaded active exited    CoreOS Configure Chrony Based On The Platform
  coreos-printk-quiet.service                                                               loaded active exited    CoreOS: Set printk To Level 4 (warn)
```

```
$ findmnt -J /
{
   "filesystems": [
      {
         "target": "/",
         "source": "composefs",
         "fstype": "overlay",
         "options": "ro,relatime,seclabel,lowerdir+=/run/ostree/.private/cfsroot-lower,datadir+=/sysroot/ostree/repo/objects,redirect_dir=on,metacopy=on"
      }
   ]
}
```

```
$ losetup
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                                                                                  DIO LOG-SEC
/dev/loop0         0      0         1  1 /sysroot/ostree/deploy/fedora-coreos/deploy/7045de9840da932191086af37d4b21d18afaff6b28bd96457e40da362e12b676.0/.ostree.cfs   1    4096
```

```
$ sudo mount -oloop /sysroot/ostree/deploy/fedora-coreos/deploy/7045de9840da932191086af37d4b21d18afaff6b28bd96457e40da362e12b676.0/.ostree.cfs /mnt
mount: /var/mnt: WARNING: source write-protected, mounted read-only.
```

```
$ sudo getfattr -d -m - /mnt/usr/share/containers/storage.conf
getfattr: Removing leading '/' from absolute path names
# file: mnt/usr/share/containers/storage.conf
security.selinux="system_u:object_r:usr_t:s0"
trusted.overlay.metacopy=""
trusted.overlay.redirect="/65/d4629a8032125b9a43b66d9c1633af34ab953ab1dd30dadfac769c925b03bb.file"
```

```
$ head -n 10 /sysroot/ostree/repo/objects/65/d4629a8032125b9a43b66d9c1633af34ab953ab1dd30dadfac769c925b03bb.file
# This file is the configuration file for all tools
# that use the containers/storage library. The storage.conf file
# overrides all other storage.conf files. Container engines using the
# container/storage library do not inherit fields from other storage.conf
# files.
#
#  Note: The storage.conf file overrides other storage.conf files based on this precedence:
#      /usr/containers/storage.conf
#      /etc/containers/storage.conf
#      $HOME/.config/containers/storage.conf
```

```
$ rpm-ostree status
State: idle
AutomaticUpdatesDriver: Zincati
  DriverState: active; periodically polling for updates (last checked Wed 2024-12-11 05:42:47 UTC)
Deployments:
● fedora:fedora/x86_64/coreos/stable
                  Version: 41.20241109.3.0 (2024-11-25T02:09:37Z)
               BaseCommit: 870e2c8fd02e81652b30bc9a33b5da9d47de66c8bc2bae0a3739ecf38f652660
             GPGSignature: Valid signature by 466CF2D8B60BC3057AA9453ED0622462E99D6AD1
          LayeredPackages: binutils

  fedora:fedora/x86_64/coreos/stable
                  Version: 41.20241109.3.0 (2024-11-25T02:09:37Z)
                   Commit: 870e2c8fd02e81652b30bc9a33b5da9d47de66c8bc2bae0a3739ecf38f652660
             GPGSignature: Valid signature by 466CF2D8B60BC3057AA9453ED0622462E99D6AD1
```

```
$ sudo bootc status
No staged image present
Current booted state is native ostree
Current rollback state is native ostree
```

# composefsの歴史

composefsは、新規のin-kernelな独自ファイルシステムという形で、2022年11月にLKMLに投稿されたRFC patchが起源となります。この後議論を重ねながらv2, v3とパッチが更新されますが、どちらかというとあまりカーネルコミュニティからの賛同は得られませんでした。むしろ新しいファイルシステムを作るよりも、似た機能を持つ既存の仕組み (overlayfs、erofs等) を改良する方がよいのではないか、という意見が出ました。

https://lore.kernel.org/lkml/cover.1669631086.git.alexl@redhat.com/

https://lore.kernel.org/lkml/cover.1673623253.git.alexl@redhat.com/

https://lore.kernel.org/lkml/cover.1674227308.git.alexl@redhat.com/

最終的に、2023年のLSFMM/BPF Summitを経て、composefsはカーネル内の新規のファイルシステムではなく、「メタデータやディレクトリツリーをerofsのイメージとし、ファイルデータをcontent-addressedなオブジェクトファイルとして、それらをoverlayfsで組み合わせる」という方向に方針転換することになりました。

その後、composefsを実現するためにいくつかの機能がoverlayfsに追加されました。代表的なものとしては
- overlayfsでmetadata only layerとdata only lower layerを持てるようにする[^5]
- overlayfsでfs-verityのサポート[^6]
があります。

[^5]: https://lore.kernel.org/all/20230427130539.2798797-1-amir73il@gmail.com/
[^6]: https://lore.kernel.org/linux-unionfs/cover.1687345663.git.alexl@redhat.com/

議論の大まかな流れは、LWNの以下の記事を順に読むとわかりやすいかもしれません。

https://lwn.net/Articles/917097/

https://lwn.net/Articles/922851/

https://lwn.net/Articles/933616/

# 参考文献

- composefsのリポジトリ
  - https://github.com/containers/composefs
- Alexander Larsson (composefsの主要開発者の一人、LKMLにパッチを投げた人) のブログ記事
  - [Using Composefs in OSTree](https://blogs.gnome.org/alexl/2022/06/02/using-composefs-in-ostree/)
  - [Composefs state of the union](https://blogs.gnome.org/alexl/2023/07/11/composefs-state-of-the-union/)
  - [Announcing composefs 1.0](https://blogs.gnome.org/alexl/2023/09/26/announcing-composefs-1-0/)
- Giuseppe Scrivano (composefsの開発者の一人、LKMLでの議論に参加している人) のブログ記事
  - [composefs - a file system for container images](https://www.scrivano.org/posts/2021-10-26-compose-fs/)

