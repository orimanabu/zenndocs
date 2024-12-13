---
title: "コンテナストレージやイメージベースのOSに特化したファイルシステムcomposefsとは"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---
# はじめに

本記事は、[OpenShift Advent Calendar 2024](https://qiita.com/advent-calendar/2024/openshift)の12/11のエントリーで、[composefs](https://github.com/containers/composefs)というコンテナ環境向けのファイルシステムを紹介します。

composefsは現在のところ、コンテナストレージおよび[OSTree](https://ostreedev.github.io/ostree/)ベースのOSイメージのリポジトリとしてのユースケースが考えられています。OSTreeを使ったLinuxディストリビューションとしては、[Fedora CoreOS](https://fedoraproject.org/coreos/)や[Fedora Atomic Desktop](https://fedoraproject.org/atomic-desktops/)、[Universal Blue](https://universal-blue.org/)プロジェクトの[派生ディストリビューション](https://universal-blue.org/#images)、[bootc](https://containers.github.io/bootc/)を使ったブータブルコンテナの環境、等が挙げられます[^1]。

bootcについては以前ブログ記事を書きました。

https://zenn.dev/orimanabu/articles/try-rhel-image-mode

[^1]: CoreOS、Atomic Desktop、bootcの違いについては、こちらをご参照ください: https://docs.fedoraproject.org/en-US/bootc/linux-desktops/ https://docs.fedoraproject.org/en-US/bootc/fedora-coreos/

先日のKubeCon NAで、[Podmanをはじめとするコンテナ関連ツール群をCNCFに寄贈する旨の発表](https://www.redhat.com/en/blog/red-hat-contribute-comprehensive-container-tools-collection-cloud-native-computing-foundation)がありましたが、その中にcomposefsとbootcも含まれています。これらのツールについてもCNCFのプロジェクトとして、より広いコミュニティに使ってほしい、という気持ちが込められているのではないかと想像します。

なんでOpenShiftとは直接関係のないcomposefsをAdvent Calendarのネタにしているかといいますと... OpenShiftのノードは「RHEL CoreOS」というLinuxディストリビューションを使っているのですが、そのアップストリームであるFedora CoreOSでは、すでにcomposefsが使われているからです。というわけで、それほど遠くない将来、OpenShiftに入ってくる(かもしれない)機能[^2]ということで、ご容赦ください... 

[^2]: 外から見えるJIRAチケットありました https://issues.redhat.com/browse/COS-2963

# composefsの機能

composefsは、いくつかの特徴を持つ新しいファイルシステムです。

GitHubリポジトリのREADMEには

> "The reliability of disk images, the flexibility of files"

とあります。ディスクイメージはtar/zipと比べて、マウントできる、dm-verityを使って改ざんを検知できる等、カーネルと連携しやすいという特徴があります。一方で、必要以上のストレージ容量が必要だったり、取り回しの柔軟さに欠ける傾向があります。composefsは、ファイルベースの実装で柔軟さを維持しつつ、改ざん検知やカーネルによるマウントをサポートするファイルシステムです。

典型的なファイルシステムはブロックデバイスをマウントしますが、composefsはext4やxfsといった通常のファイルシステム上のファイルの集合をバックエンドに持ちます。別の言い方をすると、「バックエンドとなる通常のファイルの集合 (＋ファイルツリー/メタデータの情報) に対してcomposefsマウントする」という、ある種のループバックマウントをするファイルシステムという見方もできるかもしれません。

バックエンドとなるファイル群の置き場は、「content-addressed object store」(もしくは単に「オブジェクトストア」)と呼ばれます。content addressedという言葉は、ファイルの内容からファイルを指定できる、という気持ちが込められています。具体的には、バックエンドのファイルは、ファイルの中身のSHA256のハッシュ値がファイル名となります。ネットワーク機器には搭載されているCAM (Content Addressable Memory, MACテーブルやルーティングテーブルの検索をハードウェアで高速処理するやつ) が想起される言葉ですね。

composefsは、erofs、overlayfs、fs-verityといったカーネルの機能を組み合わせて使います。具体的には、

- 通常のext4やxfsといったファイルシステム上にオブジェクトストアを作り、そこにSHA256のハッシュ値をファイル名とするファイルコンテンツを配置する
- ファイルシステムのファイルツリーやメタデータは、erofsのイメージファイルに入れる
- erofsのイメージファイルをmetadata only layer、オブジェクトストアをdata only layerとして、overlayfsで重ね合わせてマウントする

という動きになります。ファイルのメタデータとデータを別のファイルシステムに持ち、overlayfsで組み合わせる、という機能は、composefsのために開発されました。

composefsは、ファイルデータのオブジェクトストアと、ファイルツリー/メタデータのイメージを別に持つことで、

- 複数のファイルシステムで同じ中身を持つファイルがある場合は、ファイルデータをcontent-addressedなオブジェクトストアに1個持つだけで済む (ファイル単位の重複排除)
- ひとつのオブジェクトファイルを、異なるメタデータを持つファイルとして複数のファイルシステムに見せることができる

という特徴を持ちます。

composefsはfs-verityを使ってファイルシステム全体の改ざん検知ができます。fs-verityはLinuxカーネルが持つファイル単位の改ざん検知を行う仕組みです。fs-verityでは、改ざんを検知するとファイルの読み込みが失敗します (具体的にはread(2)時はEIOを、mmap(2)時はSIGBUSを返します[^3])。fs-verity自体はファイルのデータ(中身)にしか関与しません。つまりfs-verytyではメタデータの変更は検知できません。しかしcomposefsの場合は、ファイルシステムのメタデータ/ディレクトリツリーをEROFSのイメージとして持つため、このイメージファイルに対してもfs-verityのダイジェストを確認することで、メタデータを含めたファイルシステム全体の改ざん検知ができます。

[^3]: https://docs.kernel.org/filesystems/fsverity.html#accessing-verity-files

fs-verityを使用するには、ファイルシステムがサポートしている必要があります。Linuxカーネルv6.13-rc2で確認した限りでは、f2ff, ext4, btrfsがfs-verityをサポートしているようです。RHEL系のディストリビューションのデフォルトであるxfsはまだfs-verityをサポートしていませんのでご注意ください[^4]。

[^4]: xfsのfs-verityサポートは、まだupstreamで議論中です。https://lore.kernel.org/all/20240612190644.GA3271526@frogsfrogsfrogs/

```
$ grep -lr FS_IOC_ENABLE_VERITY fs
fs/f2fs/file.c
fs/f2fs/verity.c
fs/btrfs/ioctl.c
fs/ext4/inode.c
fs/ext4/verity.c
fs/ext4/ioctl.c
fs/fuse/ioctl.c
fs/verity/enable.c
fs/verity/signature.c
```

現在利用できるコンテナイメージの署名検証は仕組みは、イメージをダウンロードしたときやコンテナ実行時のタイミングで検証するものがほとんどだと思います。composefsの改ざん検知の仕組みは、コンテナ実行中にファイルを改ざんされたとしても検知することができます(fs-verity有効時はファイルが改ざんされるとread自体が失敗するため)。

ファイルの完全性(integrity)をチェックできることは、高い信頼性を要するシステムでは必須の機能です。Red HatはRHIVOSという車載Linuxディストリビューションを開発していますが、その中ではOSTreeベースのファイルシステムを使っています[^5]。ここにcomposefsをかぶせることで、runtime integrityを確保することができます。[UKI (Unified Kernel Image)](https://github.com/uapi-group/specifications/blob/main/specs/unified_kernel_image.md)を使ってSecure Bootをし、fs-verityを有効にしたcomposefs+ostreeのファイルシステムで起動することで、ブートローダー、カーネル、initramfs、カーネル起動時のコマンドライン、ファイルシステムの全てをセキュアな状態にすることができます(Secure Bootだけだと、ブートローダー、カーネル、カーネルモジュールしか保護の対象になりません)。詳細は以下の講演をご参照ください。

https://cfp.all-systems-go.io/all-systems-go-2024/talk/HVEZQQ/

[^5]: https://www.youtube.com/watch?v=zx-W2pAq1LE&t=55s

余談ですが、SteamOSのOSアップデートの仕組みで使っている[RAUC](https://github.com/rauc/rauc)というOSSプロジェクトでも、composefsを使うことになったようです[^6]。

https://cfp.all-systems-go.io/all-systems-go-2024/talk/3DKX9V/

[^6]: https://github.com/rauc/rauc/pull/1500

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

mkcomposefsコマンドを実行します。`--digest-store` オプションでオブジェクトストアを指定し、コマンドライン引数でcomposefsの元となるディレクトリと、メタデータを入れるerofsイメージファイル名を指定します。

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
(erofsのマウントポイントは、overlayfsマウントが成功したらunmountしてrmdirしているので[^7]、findmntで見てもerofsの情報は出てきません)

[^7]: https://github.com/containers/composefs/blob/ed3ee0d4530e624df2dbf78977651ce84abb14a2/libcomposefs/lcfs-mount.c#L618

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

ちなみにオブジェクトストアのファイル名となるSHA256ハッシュ値(例えばfoo.txtのデータである `85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a`)は、`fsverity digest` の値です。fs-verityのハッシュ値は、ファイルを固定サイズのブロックにわけて、各ブロックからSHA256ハッシュ値を計算し、Merkle treeというデータ構造に基づいて全体のハッシュ値を計算します[^8]。

[^8]: Merkle treeを使うと、大きいブロックデバイス/ファイルの一部分の改ざんを確認する際に、全体のハッシュ値を計算しなくてもすむため、特にブロックデバイスの改ざん検知をするdm-verityで有効な手法です。dm-verityでのMerkle treeについては https://www.starlab.io/blog/dm-verity-in-embedded-device-security https://archive.fosdem.org/2023/schedule/event/image_linux_secureboot_dmverity/  をご参照ください

```
$ fsverity digest /mnt/composefs/foo.txt
sha256:85d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a /mnt/composefs/foo.txt
```

ファイルサイズが64バイト未満の場合は、オブジェクトストアにリダイレクトせず、erofsのイメージ内に直接ファイルデータを埋め込みます。上記の例で、オブジェクトストア内にファイルが2個しかなく、erofsイメージ内のtestdataの拡張属性 `trusted.overlay.redirect` が設定されていなかったのはそのためです。

fs verityの動きを確認するため、一度アンマウントしてオブジェクトストアのファイルを意図的に変更してみます。

```
$ sudo umount /mnt/composefs
$ sed -i -e 's/foo/FOO/' objects/85/d600d462f5c3738b55c3ebf570c31263353dc6aa35448c6a8f9aa519429c8a
$ sudo mount -t composefs -o basedir=objects example.cfs /mnt/composefs
$ cat /mnt/composefs/foo.txt
FOO.txt____________________________________________________________
```

マウントオプションに `verity=on` をつけると、ファイルにアクセスした際に改ざん検知を行います。foo.txtの中身を変えているので、catするとEIOが返ります。

```
$ sudo umount /mnt/composefs
$ sudo mount -t composefs -o basedir=objects,verity=on example.cfs /mnt/composefs
$ cat /mnt/composefs/foo.txt
cat: /mnt/composefs/foo.txt: Input/output error
```

ファイルシステム全体で改ざん検知をするには、erofsイメージをfsverity enableします。後のテストのために、ハッシュ値を確認しておきます。

```
$ fsverity enable example.cfs
$ fsverity measure example.cfs
sha256:0fe266a6f4c632c168b70267bd7fc4634be62c46d5730641782184b18e7ce4e6 example.cfs
```

マウントオプションに `digest=ハッシュ値` をつけると、erofsイメージファイルの改ざん検知も行います。

```
$ sudo mount -t composefs -o basedir=objects,verity=on,digest=0fe266a6f4c632c168b70267bd7fc4634be62c46d5730641782184b18e7ce4e6 example.cfs /mnt/composefs
```

テストのため、fsverity enableしたexample.cfsを消してもう一度作成し、同じマウントコマンドを実行してみます。作り直したexample.cfsを使って(fsverity enableせずに)同じマウントオプションでマウントしようとすると、ハッシュ値の確認ができないためマウント自体が失敗します。

```
$ sudo mount -t composefs -o basedir=objects,verity=on,digest=0fe266a6f4c632c168b70267bd7fc4634be62c46d5730641782184b18e7ce4e6 example.cfs /mnt/composefs
mount.composefs: Failed to mount composefs /data/composefs/example.cfs: Image has no fs-verity
```

まとめると、composefsを作ると

- 元になるディレクトリ以下のファイルは、その中身のハッシュ値をファイル名としたファイルとして、オブジェクトストア内にコピーされる (細かいことを言うと、reflinkが使えるファイルシステムの場合はreflinkが作られます)
- 元のディレクトリツリーおよびメタデータの情報は、mkcomposefsで生成されるerofsイメージに格納されます。このとき、各ファイルの拡張属性に、対応するオブジェクトストア内のファイルのパスが入ります。
- ファイルサイズが小さい場合は、オブジェクトストアにリダイレクトせず、erofsイメージにデータを直接埋め込みます

のようになります。

# Podmanからcomposefsを使う

想定ユースケースその1、コンテナストレージです。

Podmanでcomposefsを使う場合、現時点ではrootlessはサポートされません[^9]。rootfulで実行する必要があります。

[^9]: https://github.com/containers/storage/blob/73af2c64286e8cf23e3dada7b6115df8b7a3a391/drivers/overlay/overlay.go#L373-L375 たぶん問題はerofsのイメージをループバックマウントしているところだと思われます。ループバックマウントってnamespace対応していないとか、root権限が必要とか、いろいろとrootlessに厳しい状況なので...

## 準備

storage.confの `[storage.options.overlay]` セクションで `use_composefs = "true"` を設定します。

また、composefsはzstd:chunksと一緒に使うことが前提になっているので、zstd:chunksの設定もしておきます。
具体的には、`[storage.options]` セクションで

```
pull_options = {enable_partial_images = "true", use_hard_links = "true", ostree_repos=""}
```
を設定します。

composefsはファイル単位の重複排除の機能を持ちますが(オブジェクトストアを共有した場合)、Podmanからcomposefsを使う場合、今の実装ではレイヤーごとにerofsのメタデータイメージとオブジェクトストアを作ります。つまり、重複排除の機能としてはcomposefsではなくzstd:chunkedの仕組みを使います。

composefsとzstd:chunksを組み合わせて使う場合は、ストレージおよびメモリの使用量を効率化するため、zstd:chunksの重複排除にハードリンクを使う (`use_hard_links = "true"`) のがお薦めです。
zstd:chunksでハードリンクを使う場合のいくつか注意点[^10]は、copmosefsと一緒に使う場合は気にしなくても大丈夫です (メタデータがerofsイメージに保存されるため)。

[^10]: 詳細は https://zenn.dev/orimanabu/articles/zstd-chunked-intro をご参照ください。

設定の際は、

- 設定値は `"true"` (文字列) です
- 現時点では、storage.confに関しては `/etc/containers/storage.conf.d` を使った設定はサポートされていないので[^11]、`/usr/share/containers/storage.conf` を `/etc/containers/storage.conf` にコピーしてそれを編集します。

の2点に注意してください。

[^11]: PRは出ています https://github.com/containers/storage/pull/1885

storage.confを編集したら、`sudo podman system reset` を実行しておきます。

## コンテナイメージ

Podmanでcomposefsを使用するためには、メディアタイプが `application/vnd.oci.image.layer.v1.tar+zstd` のコンテナイメージを使う必要があります。Podmanでコンテナイメージをビルドして、`podman push` 時に `--compression-format=zstd:chunks` オプションをつけると、zstd:chunksフォーマットのコンテナイメージをpushできます。また、storage.confで `` の設定をしておくと、通常のtar+gzipフォーマットのコンテナイメージを自動的にzstd:chunksフォーマットに変換してくれます (時間がかかるので設定する際はご注意ください)。以下の例では `quay.io/manabu.ori/myhttpd` を使用します。このイメージはzstdで圧縮したものになっています。

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

ループバックマウントの様子を確認します。使用しているコンテナイメージは4つのレイヤーからなっているため、ループバックデバイスも4つできています。

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

overlayfsマウントのオプションのlowerdirを見やすく整形してみましょう。

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

コロンが2個連続している部分(`::`)の右側のレイヤーが data only lower layers で、この中のファイルはcomposefsにおけるオブジェクトストアであることを表しています[^12]。

composefsを使用しない場合、rootful使用時のコンテナストレージのディレクトリレイアウトは、`/var/lib/containers/storage/overlay/<ハッシュ値>diff` 以下にコンテナイメージのtarballを展開します。一方、composefsを使用する場合は、

- `/var/lib/containers/storage/overlay/<ハッシュ値>composefs-data/composefs.blob` : erofsのイメージファイル
- `/var/lib/containers/storage/overlay/<ハッシュ値>diff/` : オブジェクトストア

のようになります。今使っているコンテナイメージは4つのレイヤーからなりますが、そのうちの一つを覗いてみましょう。

[^12]: https://docs.kernel.org/filesystems/overlayfs.html#data-only-lower-layers

erofsのイメージを見ると、拡張属性 `trusted.overlay.redirect` にオブジェクトストアのパスが書かれています。

```
# mount -oloop /var/lib/containers/storage/overlay/ed76af94e8a065c5e3d1bf6a8db625cee90947f69624b380eb4a5beb1082d414/composefs-data/composefs.blob /mnt/tmp
# cat /mnt/tmp/var/www/html/index.html
# getfattr -d -m - /mnt/tmp/var/www/html/index.html
getfattr: Removing leading '/' from absolute path names
# file: mnt/tmp/var/www/html/index.html
trusted.overlay.metacopy=0sACQAAeG6n0gwFNmWFkJWaUL9QlUwHYFLULh0Z7qRFqFgm3iQ
trusted.overlay.redirect="/f1/f2463d6c783031a0b34d4c545b4383a1d24a3c6efefc33cc0cd6ca339b0dea"
```

オブジェクトストアのパスをたどると、確かにindex.htmlでした。

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

- bootcでOS領域をコンテナイメージにする
- Fedora CoreOS

の2つの場合を見てみます (ほとんど同じです)。

## bootc

libvirtで仮想マシンを作成します。作業の流れとしては

- `fedora-bootc`というコンテナイメージ(kernel、initramfs、systemd等が入っています)をベースにしたコンテナイメージを作る
- そのコンテナイメージを元に、`bootc-image-builder` を使ってqcow2の仮想マシンイメージを作成する
- virt-installで仮想マシンを作成・起動する

という感じです。

細かい手順は下記をご参照ください。

https://docs.fedoraproject.org/en-US/bootc/getting-started/

[podman-bootc](https://github.com/containers/podman-bootc.git)というヘルパーコマンドがあって、これを使って

```
podman-bootc run --filesystem=xfs quay.io/manabu.ori/fedora-bootc-test:41
```

みたいな感じで実行すると、上記の「コンテナイメージからqcow2の仮想マシンイメージを作って仮想マシンを起動してsshログインする」を一気に行うことができます。

sshログインして、/etc/os-releaseを確認します。

```
# cat /etc/os-release
NAME="Fedora Linux"
VERSION="41.20241210.0 (Forty One)"
...
OSTREE_VERSION='41.20241210.0'
```

カーネルの起動オプションにostreeのcommitを指定しています。

```
# cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/boot/ostree/default-fb1fef48e415876a4a43c311b1ef3c5ecfea7094e0befb2518c631fca55deeb4/vmlinuz-6.11.10-300.fc41.x86_64 root=UUID=da43332a-c607-45a9-808e-1eaa0ec68ba5 rw ostree=/ostree/boot.1/default/fb1fef48e415876a4a43c311b1ef3c5ecfea7094e0befb2518c631fca55deeb4/0
```

定期的にOS領域のコンテナイメージの更新をチェックし、更新があればpullするサービス `bootc-fetch-apply-updates.timer` が起動しています (手元の環境の事情でfailedになっていますが...)。

```
# systemctl | grep -E '(coreos|bootc)-'
● bootc-fetch-apply-updates.service                                            loaded failed failed    Apply bootc updates
  bootc-fetch-apply-updates.timer                                              loaded active waiting   Apply bootc updates
```

ルートファイルシステムをcomposefsでマウントしています。

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

## Fedora CoreOS

下記からqcow2イメージをダウンロードして、適当にignitionファイルを書いて起動します。

https://fedoraproject.org/coreos/download?stream=stable

virt-installはこんな感じです。

```
sudo virt-install \
--name=fcos41 \
--vcpus=2 \
--memory=4096 \
--os-variant=fedora-coreos-stable \
--import \
--graphics=none \
--disk="size=20,backing_store=/var/lib/libvirt/images/fcos41.qcow2" \
--network network=default \
--qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=$(pwd)test.ign"
```

ログインして/etc/os-releaseと/proc/cmdlineを見てみます。

```
$ cat /etc/os-release
NAME="Fedora Linux"
VERSION="41.20241109.3.0 (CoreOS)"
...
VARIANT="CoreOS"
VARIANT_ID=coreos
OSTREE_VERSION='41.20241109.3.0'
```

```
$ cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/boot/ostree/fedora-coreos-717f957fb4680546cce36bc1c5e633abdbf5e4ecb6e99e5665df3c00a56088fa/vmlinuz-6.11.6-300.fc41.x86_64 rw mitigations=auto,nosmt ostree=/ostree/boot.0/fedora-coreos/717f957fb4680546cce36bc1c5e633abdbf5e4ecb6e99e5665df3c00a56088fa/0 ignition.platform.id=qemu console=tty0 console=ttyS0,115200n8 root=UUID=4828f807-1e7f-419a-88f2-31df1a736fa1 rw rootflags=prjquota boot=UUID=a6192326-87ec-42a5-9653-ae5424995744
```

ルートファイルシステムをcomposefsでマウントしています。

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

# composefsの歴史

composefsは、新規のin-kernelな独自ファイルシステムという形で、2022年11月にLKMLに投稿されたRFC patchが起源となります。この後議論を重ねながらv2, v3とパッチが更新されますが、どちらかというとあまりカーネルコミュニティからの賛同は得られませんでした。むしろ新しいファイルシステムを作るよりも、似た機能を持つ既存の仕組み (overlayfs、erofs等) を改良する方がよいのではないか、という意見が出ました。

https://lore.kernel.org/lkml/cover.1669631086.git.alexl@redhat.com/

https://lore.kernel.org/lkml/cover.1673623253.git.alexl@redhat.com/

https://lore.kernel.org/lkml/cover.1674227308.git.alexl@redhat.com/

最終的に、2023年のLSFMM/BPF Summitでのface to faceの議論を経て、composefsはカーネル内の新規のファイルシステムではなく、「メタデータやディレクトリツリーをerofsのイメージとし、ファイルデータをcontent-addressedなオブジェクトファイルとして、それらをoverlayfsで組み合わせる」という方向に方針転換することになりました。

その後、composefsを実現するためにいくつかの機能がoverlayfsに追加されました。代表的なものとしては

- overlayfsでmetadata only layerとdata only lower layerを持てるようにする[^13]
- overlayfsでfs-verityのサポートする[^14]

があります。

[^13]: https://lore.kernel.org/all/20230427130539.2798797-1-amir73il@gmail.com/
[^14]: https://lore.kernel.org/linux-unionfs/cover.1687345663.git.alexl@redhat.com/

議論の大まかな流れは、LWNの以下の記事を順に読むとわかりやすいかもしれません。

https://lwn.net/Articles/917097/

https://lwn.net/Articles/922851/

https://lwn.net/Articles/933616/

# 参考文献

- composefsのリポジトリ
  - https://github.com/containers/composefs
- FOSDEM2024
  - [Composefs and containers](https://archive.fosdem.org/2024/schedule/event/fosdem-2024-3250-composefs-and-containers/)
- Alexander Larsson (composefsの主要開発者の一人、LKMLにパッチを投げた人) のブログ記事
  - [Using Composefs in OSTree](https://blogs.gnome.org/alexl/2022/06/02/using-composefs-in-ostree/)
  - [Composefs state of the union](https://blogs.gnome.org/alexl/2023/07/11/composefs-state-of-the-union/)
  - [Announcing composefs 1.0](https://blogs.gnome.org/alexl/2023/09/26/announcing-composefs-1-0/)
- Giuseppe Scrivano (composefsの開発者の一人、LKMLでの議論に参加している人) のブログ記事
  - [composefs - a file system for container images](https://www.scrivano.org/posts/2021-10-26-compose-fs/)

