---
title: "Image mode for Red Hat Enterprise Linuxの中身を見てみる"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rpm-ostree", "ostree", "bootc", "Fedora", "RHEL"]
published: true
---
# はじめに

Red Hat Summit 2024でImage mode for Red Hat Enterprise Linuxが発表されました。Red Hatからいくつか関連情報が出ていますが、どれも抽象的でいまいちよくわかりません。

- [Red Hat Reimagines Enterprise Linux for the AI Future with Image Mode for Red Hat Enterprise Linux](https://www.redhat.com/en/about/press-releases/red-hat-reimagines-enterprise-linux-ai-future-image-mode-red-hat-enterprise-linux) (プレスリリース)
- [Experience the AI-ready OS with image mode for Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux/image-mode) (製品ページ)
- [Image Mode for RHEL を使用したオペレーティングシステムの構築、デプロイ、管理](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/9/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index) (製品ドキュメント)

Publickeyでも記事が出ています。

- [コンテナイメージなのにブート可能な新技術による「Image mode for Red Hat Enterprise Linux」、Red Hatが発表。レジストリなどのコンテナ関連ツールがそのまま利用可能](https://www.publickey1.jp/blog/24/image_mode_for_red_hat_enterprise_linuxred_hat.html)

「コンテナイメージなのにブート可能」... はて？

というわけで本記事では、Image modeがどんなものか調べるために、[Image mode for Red Hat Enterprise Linux: A quick start guide](https://www.redhat.com/en/blog/image-mode-red-hat-enterprise-linux-quick-start-guide) (以下「クイックガイド」)に載っている手順にしたがって実際に動かしてみました。クイックガイドではRHELを使った手順が紹介されていますが、手元ではそのままでは動かなかった部分もあったため、本記事ではFedora/CentOS Stream 9のイメージを使いました。仕組み自体はImage mode for RHELと同じです。

以下は2024年5月9日時点の検証結果です。まだもりもりと開発が続いているようなので、今後は変わるところもあるかもしれません。

# 結論を最初に書いておくと...

Image mode for RHELをまとめると、以下になります。

- Image mode for RHELは、OSのファイルシステムツリーをレイヤー化したコンテナイメージとして配布します

  - OSとして起動するための最低限のファイルを含む必要があります(カーネル、initramfs、systemd等)

- Image modeのコンテナイメージは、(もちろん)コンテナとして起動することができます。

- Image modeのコンテナイメージをOSとして起動する場合は、対象とするプラットフォーム向けに初期インストール用のイメージ変換等が必要な場合があります。コンテナイメージに入っているのはファイルシステムを再現するための情報のみなので、それよりも下の層(ディスクのパーティショニング、ブートローダーのインストール等)については、手動もしくはイメージ変換ツール(bootc-image-builder)の中で行います。

  - ベアメタルの場合は、Kickstartの `ostreecontainer` 命令で直接コンテナイメージを指定できます。パーティション作成やブートローダーのインストールはKickstartにいつもどおり手動で書く必要があります。
  - KVM仮想マシンの場合は、コンテナイメージからqcow2イメージに変換して、それを使ってVMを起動します。パーティションの作成やブートローダーのインストールはイメージ変換ツールがやってくれます。
  - AWSのEC2インスタンスの場合も(以下略)

- この変換作業が必要なのは、初期インストール時に一度だけです。OSとして起動した後のOSアップデートは、コンテナイメージを更新後push → OS上でpullして展開して再起動、するだけです

  - 初期設定ではsystemd timerユニットで定期的に更新チェックしているため、OSのコンテナイメージを更新してレジストリにpushしておくと、Image modeのOSは自動的にそれをpullして適用して再起動します。定期チェックを停止して任意のタイミングで手動アップデートすることもできます。

- ostreeの仕組みを使ってOSのファイルシステムを構成しているため、次の特徴があります

  - 複数のイメージを展開しておき、起動時に切り替えることができます (つまり前のイメージにロールバックできます)
  - `/boot`, `/etc`, `/var` 以外はread onlyマウントしているため、万が一侵入されたとしても被害を低減することができます
  - OS (のファイルシステムイメージ) にコンテナイメージとしての可搬性を持たせることができますので、CI/CD等の「テストしたときと同じ全く環境をデプロイする」というユースケースで活躍できる...かもしれません

- コンテナイメージは多くの場合、rpmパッケージから作成すると思います。その場合、Image modeで起動した場合でも元となるディストリビューションと同じバイナリが動きます (例えばImage mode for RHELであれば、通常のRHELと同じカーネルやコマンドのバイナリを使います)。

Image modeを実現するための要素技術は以下です。

- [ostree](https://github.com/ostreedev/ostree): (複数のファイルシステムツリーをGitリポジトリ的に管理して切り替えられるようにする)

  - ostreeについてはこちらの資料がとても参考になります: [OSTree: OSイメージとパッケージシステムの間にGitのアプローチを](https://www.slideshare.net/i_yudai/ostree-osgit)

- [rpm-ostree](https://github.com/coreos/rpm-ostree): (ostreeを構成するファイル群を、rpmパッケージを使って構成する)
- [bootc](https://github.com/containers/bootc): (OCIコンテナイメージを使ってOSをアップデートできるようにする)

  - bootcはRustで書かれています。

- [bootc-image-builder](https://github.com/osbuild/bootc-image-builder): (コンテナイメージから各種プラットフォーム用のOSイメージに変換する)

さて、CoreOS (Container LinuxではなくRed Hat買収後の方) の中身をご存知の方は、Image modeの特徴はCoreOSととてもよく似ていると思われたのではないでしょうか。実際、ほとんどの部分はCoreOSと共通です。CoreOSにbootcの仕組みを合体させたものがImage modeと思ってもよいかもしれません。

# コンテナイメージの作成

OS起動およびImage modeに必要な最低限のツールを含んだイメージ(以下bootcイメージ)が用意されているので、それをベースに自分のコンテナイメージを作成します。

クイックガイドではRHEL 9ベースのbootcイメージをベースにしていますが、本記事ではサブスクリプション契約の必要がないFedora 40ベースのbootcイメージを使いました。

```
podman pull quay.io/fedora/fedora-bootc:40
```

(以外とレイヤーが多いのが印象的です)

これをベースにして、以下のようなコンテナイメージを作成します。

```
FROM quay.io/fedora/fedora-bootc:40

#install the lamp components
RUN dnf install -y httpd mariadb mariadb-server php-fpm php-mysqlnd && dnf clean all

#start the services automatically on boot
RUN systemctl enable httpd mariadb php-fpm

#create an awe inspiring home page!
RUN echo '<h1 style="text-align:center;">Welcome to image mode for RHEL</h1> <?php phpinfo(); ?>' >> /var/www/html/index.php
```

クイックガイドでは `Containerfile` という名前のファイルに記述していますが、ファイル名は `Dockerfile` でも構いません。

```
podman build -f Dockerfile -t quay.io/manabu.ori/lamp-bootc:latest
```

テストのため、コンテナとして起動してみます。

```
podman run -d --rm --name lamp -p 8080:80 quay.io/manabu.ori/lamp-bootc:latest /sbin/init
```

アクセスできることを確認します。

```
$ curl -IL localhost:8080
HTTP/1.1 200 OK
Date: Thu, 09 May 2024 11:08:25 GMT
Server: Apache/2.4.59 (Fedora Linux)
X-Powered-By: PHP/8.3.6
Content-Type: text/html; charset=UTF-8
```

Dockerfileに記載したとおり、initがPID 1で、その下にApache httpdやMariaDBが動いています。

```
$ podman exec lamp ps axjf
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
      0     526       0       0 ?             -1 R        0   0:00 ps axjf
      0       1       1       1 ?             -1 Rs       0   0:01 /sbin/init
      1      33      33      33 ?             -1 Ss       0   0:00 /usr/lib/systemd/systemd-journald
      1      38      38      38 ?             -1 Ss       0   0:00 /usr/lib/systemd/systemd-userdbd
     38      52      38      38 ?             -1 S        0   0:00  \_ systemd-userwork
     38      53      38      38 ?             -1 S        0   0:00  \_ systemd-userwork
     38      54      38      38 ?             -1 S        0   0:00  \_ systemd-userwork
      1     111     111     111 ?             -1 Ss      81   0:00 /usr/bin/dbus-broker-launch --scope system --audit
    111     112     111     111 ?             -1 S       81   0:00  \_ dbus-broker --log 4 --controller 9 --machine-id e74107fdcae9eb21db962afb2df63916 --max-bytes 536870912 --max-fds 4096 --max-matches 131072 --audit
      1     113     113     113 ?             -1 Ssl      0   0:00 /usr/sbin/NetworkManager --no-daemon
      1     114     114     114 ?             -1 Ss      70   0:00 avahi-daemon: registering [e74107fdcae9-5.local]
    114     132     114     114 ?             -1 S       70   0:00  \_ avahi-daemon: chroot helper
      1     120     120     120 ?             -1 Ss       0   0:00 /usr/lib/systemd/systemd-homed
      1     121     121     121 ?             -1 Ss       0   0:00 /usr/lib/systemd/systemd-logind
      1     124     124     124 ?             -1 Ssl      0   0:00 /usr/libexec/udisks2/udisksd
      1     154     154     154 ?             -1 Ss       0   0:00 /usr/lib/systemd/systemd-hostnamed
      1     160     160     160 ?             -1 Ssl      0   0:00 /usr/libexec/nm-dispatcher
      1     164     164     164 ?             -1 Ss       0   0:00 php-fpm: master process (/etc/php-fpm.conf)
    164     267     164     164 ?             -1 S       48   0:00  \_ php-fpm: pool www
    164     269     164     164 ?             -1 S       48   0:00  \_ php-fpm: pool www
    164     272     164     164 ?             -1 S       48   0:00  \_ php-fpm: pool www
    164     274     164     164 ?             -1 S       48   0:00  \_ php-fpm: pool www
    164     275     164     164 ?             -1 S       48   0:00  \_ php-fpm: pool www
      1     181     181     181 ?             -1 Ssl      0   0:00 /usr/sbin/gssproxy -D
      1     187     187     187 ?             -1 Ss       0   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
      1     203     203     203 ?             -1 Ss       0   0:00 /usr/sbin/httpd -DFOREGROUND
    203     298     203     203 ?             -1 S       48   0:00  \_ /usr/sbin/httpd -DFOREGROUND
    203     300     203     203 ?             -1 Sl      48   0:00  \_ /usr/sbin/httpd -DFOREGROUND
    203     302     203     203 ?             -1 Sl      48   0:00  \_ /usr/sbin/httpd -DFOREGROUND
    203     329     203     203 ?             -1 Sl      48   0:00  \_ /usr/sbin/httpd -DFOREGROUND
      1     221     221     221 ?             -1 Zs      27   0:00 [mariadb-prepare] <defunct>
```

コンテナを停止します。

```
podman stop lamp
```

コンテナレジストリにpushします。

```
podman login quay.io
podman push quay.io/manabu.ori/lamp-bootc:latest
```

# qcow2イメージの作成

イメージ変換はrootユーザーで実行します。

Podmanは実行ユーザーごとにコンテナストレージが異なるため、まず作成したイメージをrootでpullしておきます。

```
sudo podman image pull quay.io/manabu.ori/lamp-bootc:latest
```

bootc-image-builderのコンフィグファイルを `config.json` として作成します (tomlでも書けるようですが、クイックガイドにそってjsonにしました)。

```
{
 "blueprint": {
   "customizations": {
     "user": [
       {
         "name": "cloud-user",
         "password": "changeme",
         "key": "ssh-rsa AAAAB3Nz...",
         "groups": [
           "wheel"
         ]
       }
     ]
   }
 }
}
```

bootc-image-builder (のコンテナ) を使って、作成したコンテナイメージをqcow2イメージに変換します。

```
mkdir -p output
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    -v $(pwd)/config.json:/config.json \
    -v $(pwd)/output:/output \
    quay.io/centos-bootc/bootc-image-builder:latest \
    --type qcow2 \
    --local \
    --rootfs xfs \
    --log-level info \
    quay.io/manabu.ori/lamp-bootc:latest
```

(結構時間がかかります)

# イメージをOSとして起動

qcow2に変換したイメージをKVM仮想マシンとして起動します。

```
sudo virt-install --name lamp-bootc --memory 4096 --vcpus 2 --disk ./output/qcow2/disk.qcow2 --import --os-variant rhel9.4
```

## 動作確認

IPアドレスを調べてアクセスしてみます。

```
$ sudo virsh domifaddr lamp-bootc
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet61     52:54:00:6f:da:1c    ipv4         192.168.124.31/24
```

```
$ curl -IL 192.168.124.31
HTTP/1.1 200 OK
Date: Thu, 09 May 2024 15:29:59 GMT
Server: Apache/2.4.59 (Fedora Linux)
X-Powered-By: PHP/8.3.6
Content-Type: text/html; charset=UTF-8
```

# 仮想マシンの中を見てみる

config.jsonに設定した鍵もしくはパスワードでログインします。

```
ssh -i PRIVATE_KEY -l cloud-user 192.168.124.31
```

以下のコマンドは、Image modeで起動した仮想マシン上で実行します。

```
$ cat /etc/fedora-release
Fedora release 40 (Forty)
```

```
$ uname -r
6.8.8-300.fc40.x86_64
```

カーネルの起動パラメータで`ostree=...` によって起動するファイルシステムツリーのIDを指定しています。

```
$ cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/boot/ostree/default-7292e7ed0ab72bdf2384d177b90601f1e86c37d32aeb67c59d974adcf5300e21/vmlinuz-6.8.8-300.fc40.x86_64 root=UUID=d1eea7fc-73f9-4a87-9dd1-ad0e903b9718 rw boot=UUID=60ef6a56-5356-48d8-83ae-c6f154ccbef8 rw console=tty0 console=ttyS0 ostree=/ostree/boot.1/default/7292e7ed0ab72bdf2384d177b90601f1e86c37d32aeb67c59d974adcf5300e21/0
```

```
$ rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/manabu.ori/lamp-bootc:latest
                   Digest: sha256:6530390c80ab7cb4e4d80361b13b6eccba92d746360b0530563830884437296a
                  Version: 40.20240508.0 (2024-05-09T11:02:20Z)
```

# ファイルシステム

ostreeのリポジトリが `/sysroot` にあることがわかります。

```
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  9.8M  1 loop
vda    252:0    0   10G  0 disk
├─vda1 252:1    0    1M  0 part
├─vda2 252:2    0  501M  0 part /boot/efi
├─vda3 252:3    0    1G  0 part /boot
└─vda4 252:4    0  8.5G  0 part /var
                                /sysroot/ostree/deploy/default/var
                                /etc
                                /sysroot
```

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda4       8.5G  2.2G  6.4G  25% /sysroot
overlay         9.8M  9.8M     0 100% /
devtmpfs        4.0M     0  4.0M   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           782M  808K  782M   1% /run
tmpfs           2.0G     0  2.0G   0% /tmp
/dev/vda3       974M   83M  824M  10% /boot
/dev/vda2       501M  7.5M  494M   2% /boot/efi
tmpfs           391M  4.0K  391M   1% /run/user/1000
```

`/` が `ro` で、`/boot`, `/etc`, `/var` が `rw` でマウントされていることがわかります。

```
$ findmnt
TARGET                                 SOURCE                                                                                                          FSTYPE      OPTIONS
/                                      overlay                                                                                                         overlay     ro,relatime,seclabel,lowerdir=/run/ostree/.private/cfsroot-lower::/sysroot/ostree/repo/objects,redirect_dir=on,metacopy=on
├─/tmp                                 tmpfs                                                                                                           tmpfs       rw,nosuid,nodev,seclabel,size=2001496k,nr_inodes=1048576,inode64
├─/dev                                 devtmpfs                                                                                                        devtmpfs    rw,nosuid,seclabel,size=4096k,nr_inodes=491976,mode=755,inode64
│ ├─/dev/hugepages                     hugetlbfs                                                                                                       hugetlbfs   rw,nosuid,nodev,relatime,seclabel,pagesize=2M
│ ├─/dev/mqueue                        mqueue                                                                                                          mqueue      rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/dev/shm                           tmpfs                                                                                                           tmpfs       rw,nosuid,nodev,seclabel,inode64
│ └─/dev/pts                           devpts                                                                                                          devpts      rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000
├─/sys                                 sysfs                                                                                                           sysfs       rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/fs/selinux                    selinuxfs                                                                                                       selinuxfs   rw,nosuid,noexec,relatime
│ ├─/sys/kernel/debug                  debugfs                                                                                                         debugfs     rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/kernel/tracing                tracefs                                                                                                         tracefs     rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/fs/fuse/connections           fusectl                                                                                                         fusectl     rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security               securityfs                                                                                                      securityfs  rw,nosuid,nodev,noexec,relatime
│ ├─/sys/fs/cgroup                     cgroup2                                                                                                         cgroup2     rw,nosuid,nodev,noexec,relatime,seclabel,nsdelegate,memory_recursiveprot
│ ├─/sys/fs/pstore                     pstore                                                                                                          pstore      rw,nosuid,nodev,noexec,relatime,seclabel
│ ├─/sys/fs/bpf                        bpf                                                                                                             bpf         rw,nosuid,nodev,noexec,relatime,mode=700
│ └─/sys/kernel/config                 configfs                                                                                                        configfs    rw,nosuid,nodev,noexec,relatime
├─/proc                                proc                                                                                                            proc        rw,nosuid,nodev,noexec,relatime
│ └─/proc/sys/fs/binfmt_misc           systemd-1                                                                                                       autofs      rw,relatime,fd=36,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=2908
│   └─/proc/sys/fs/binfmt_misc         binfmt_misc                                                                                                     binfmt_misc rw,nosuid,nodev,noexec,relatime
├─/run                                 tmpfs                                                                                                           tmpfs       rw,nosuid,nodev,seclabel,size=800600k,nr_inodes=819200,mode=755,inode64
│ └─/run/user/1000                     tmpfs                                                                                                           tmpfs       rw,nosuid,nodev,relatime,seclabel,size=400296k,nr_inodes=100074,mode=700,uid=1000,gid=1000,inode64
├─/var                                 /dev/vda4[/ostree/deploy/default/var]                                                                           xfs         rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│ └─/var/lib/nfs/rpc_pipefs            sunrpc                                                                                                          rpc_pipefs  rw,relatime
├─/boot                                /dev/vda3                                                                                                       ext4        ro,relatime,seclabel
│ └─/boot/efi                          /dev/vda2                                                                                                       vfat        rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=winnt,errors=remount-ro
├─/sysroot                             /dev/vda4                                                                                                       xfs         ro,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
│ └─/sysroot/ostree/deploy/default/var /dev/vda4[/ostree/deploy/default/var]                                                                           xfs         rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
└─/etc                                 /dev/vda4[/ostree/deploy/default/deploy/c3342872193d11d77bf00a0da655280c3d5057e3963a641e816b9e300598b90e.0/etc] xfs         rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

ちなみに `/home`, `/root` はそれぞれ `/var/home`, `/var/roothome` のシンボリックリンクです。

```
$ ls -l /home /root
lrwxrwxrwx. 1 root root  8 Jan  1  1970 /home -> var/home
lrwxrwxrwx. 1 root root 12 Jan  1  1970 /root -> var/roothome
```

デフォルトで `bootc-fetch-apply-updates.timer` というtimer unitが動いています。

```
$ systemctl status bootc-fetch-apply-updates.timer
● bootc-fetch-apply-updates.timer - Apply bootc updates
     Loaded: loaded (/usr/lib/systemd/system/bootc-fetch-apply-updates.timer; disabled; preset: disabled)
     Active: active (waiting) since Thu 2024-05-09 12:48:33 UTC; 2h 53min ago
    Trigger: Thu 2024-05-09 22:15:29 UTC; 6h left
   Triggers: ● bootc-fetch-apply-updates.service
       Docs: man:bootc(8)

May 09 12:48:33 fedora systemd[1]: Started bootc-fetch-apply-updates.timer - Apply bootc updates.
```

```
$ systemctl cat bootc-fetch-apply-updates.timer
# /usr/lib/systemd/system/bootc-fetch-apply-updates.timer
[Unit]
Description=Apply bootc updates
Documentation=man:bootc(8)
ConditionPathExists=/run/ostree-booted

[Timer]
OnBootSec=1h
# This time is relatively arbitrary and obviously expected to be overridden/changed
OnUnitInactiveSec=8h
# When deploying a large number of systems, it may be beneficial to increase this value to help with load on the registry.
RandomizedDelaySec=2h

[Install]
WantedBy=timers.target
```

このtimer unitの中では `bootc update --apply` コマンドを実行しています。

```
$ cat /usr/lib/systemd/system/bootc-fetch-apply-updates.service
[Unit]
Description=Apply bootc updates
Documentation=man:bootc(8)
ConditionPathExists=/run/ostree-booted

[Service]
Type=oneshot
ExecStart=/usr/bin/bootc update --apply --quiet
```

このコマンドを手動で実行すると、実行したタイミングでOSのコンテナイメージを取得して適用後、再起動します。OS起動後のアップデートでは、初期インストール時のようなbootc-image-builderによるqcow2への変換は必要なく、コンテナイメージを直接使用できます。

timer unitによるコンテナイメージの更新→再起動時のjournaldのログの抜粋はこのような感じになります。

```
<snip>
May 09 12:44:41 fedora systemd[1]: Starting bootc-fetch-apply-updates.service - Apply bootc updates...
May 09 12:44:41 fedora kernel: SELinux:  Context system_u:object_r:invalid_bootcinstall_testlabel_t:s0 is not valid (left unmapped).
May 09 12:44:41 fedora kernel: EXT4-fs (vda3): re-mounted 60ef6a56-5356-48d8-83ae-c6f154ccbef8 r/w. Quota mode: none.
May 09 12:44:42 fedora bootc[1674]: Fetching ostree-unverified-registry:quay.io/manabu.ori/lamp-bootc:latest
May 09 12:44:46 fedora bootc[1674]: layers already present: 68; layers needed: 1 (150 bytes)
May 09 12:44:46 fedora bootc[1674]: layers already present: 68; layers needed: 1 (150 bytes)
May 09 12:45:26 fedora bootc[1674]: Image contains non-ostree compatible file paths: run: 1
May 09 12:47:22 fedora systemd[1]: Starting ostree-finalize-staged-hold.service - Hold /boot Open for OSTree Finalize Staged Deployment...
May 09 12:47:22 fedora systemd[1]: Started ostree-finalize-staged-hold.service - Hold /boot Open for OSTree Finalize Staged Deployment.
May 09 12:47:22 fedora systemd[1]: Finished ostree-finalize-staged.service - OSTree Finalize Staged Deployment.
May 09 12:47:27 fedora bootc[1674]: Queued for next boot: quay.io/manabu.ori/lamp-bootc:latest
May 09 12:47:27 fedora bootc[1674]:   Version: 40.20240508.0
May 09 12:47:27 fedora bootc[1674]:   Digest: sha256:b9be0d08af0091b171b3730b307e2c2113f314b14e633be90193c92a64b8bcad
May 09 12:47:27 fedora bootc[1674]: Total new layers: 69    Size: 868.0 MB
May 09 12:47:27 fedora bootc[1674]: Removed layers:   0     Size: 0 bytes
May 09 12:47:27 fedora bootc[1674]: Added layers:     1     Size: 150 bytes
May 09 12:47:27 fedora bootc[1674]: Rebooting system
May 09 12:47:27 fedora systemd-logind[793]: The system will reboot now!
May 09 12:47:27 fedora systemd-logind[793]: System is rebooting.
May 09 12:47:27 fedora systemd[1]: Stopping session-1.scope - Session 1 of User cloud-user...
<snip>
```

アップデート後は、ostreeのツリーが2つに増えていることがわかります (下の例ではVersionの番号が同じになっていますが、コンテナイメージのLABELか何かで正しく指定すると変えられる...はず...)。

```
$ rpm-ostree status
State: idle
Deployments:
● ostree-unverified-registry:quay.io/manabu.ori/lamp-bootc:latest
                   Digest: sha256:b9be0d08af0091b171b3730b307e2c2113f314b14e633be90193c92a64b8bcad
                  Version: 40.20240508.0 (2024-05-09T11:58:43Z)

  ostree-unverified-registry:quay.io/manabu.ori/lamp-bootc:latest
                   Digest: sha256:6530390c80ab7cb4e4d80361b13b6eccba92d746360b0530563830884437296a
                  Version: 40.20240508.0 (2024-05-09T11:02:20Z)
```

