---
title: "Rosettaはなぜ特定のVMM上の仮想マシンでないと使えないか、そしてlibkrunはどうやってそれを回避していたか"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rosetta, macos, libkrun]
published: true
---

# はじめに
Appleは、Linux用のRosettaバイナリを提供しています。
Rosettaを使うと、Apple Siliconを搭載するハードウェアのmacOSで稼働するaarch64 Linux仮想マシン上で、x86_64バイナリを実行できるようになります[^1]。

Rosettaは、実行環境がVirtualization.framework(Apple純正のVMM用フレームワーク)を使った仮想環境でないと使えません。
ですが、VMMライブラリである[libkrun](https://github.com/containers/libkrun)は、Virtualization.frameworkを使っていないにもかかわらずRosettaを使うことができます(した) (実はこの機能はrevertされており、今はできません)。

本文書は、(1) RosettaがどうやってVMMをチェックしているのか、(2) libkrunはどうやって回避したのか、の2点についてまとめたものです。

# 前提知識

macOSは、仮想化を支援するフレームワークとして[Hypervisor.framework](https://developer.apple.com/documentation/hypervisor)、[Virtualization.framework](https://developer.apple.com/documentation/virtualization)の2つを持っています。Hypervisor.frameworkは仮想化のコア機能を提供するライブラリで、Linuxにおけるkvm.koに相当します。Virtualization.frameworkはVMMを実現するためのライブラリで、LinuxにおけるQemuに相当します。

libkrunや[HVFアクセラレーションを使ったQemu](https://wiki.qemu.org/Features/HVF)は、Hypervisor.frameworkを使っています。

Virtualization.frameworkを使っているツールとしては、[UTM](https://mac.getutm.app/)や[Lima](https://lima-vm.io/)などがあります。macOS上でPodmanを実行するときも、v5.0以降ではVirtualization.frameworkを使ったLinux仮想マシンを使います。

# RosettaのVMMチェックの仕組み

「[Quick look at Rosetta on Linux](https://threedots.ovh/blog/2022/06/quick-look-at-rosetta-on-linux/)」というブログ記事によると、「rosettaを実行すると、rosettaバイナリ(`/proc/self/exe`)に謎のioctl(2)を発行して、特定の文字列が返ってくること」を確認しているようです。

...で終わるのもナニなので、実際に確認してみます。

## 環境
- ホスト: Macbook Air M2, macOS, UTM
- ゲスト: Fedora 40

```sh
ori@myfedora:~/work$ sudo dmidecode | head -n 15
# dmidecode 3.6
Getting SMBIOS data from sysfs.
SMBIOS 3.3.0 present.
Table at 0x16FCB3000.

Handle 0x0000, DMI type 1, 27 bytes
System Information
	Manufacturer: Apple Inc.
	Product Name: Apple Virtualization Generic Platform
	Version: 1
	Serial Number: Virtualization-c4e505cf-ced1-493f-9d6f-f5e46434e9d0
	UUID: cf05e5c4-d1ce-3f49-9d6f-f5e46434e9d0
	Wake-up Type: Power Switch
	SKU Number: Not Specified
	Family: Not Specified
```

```sh
ori@myfedora:~/work$ uname -r
6.8.5-301.fc40.aarch64
```

## Rosettaを使えるようにする

```sh
ori@myfedora:~/work$ sudo mount -t virtiofs rosetta /mnt/rosetta
ori@myfedora:~/work$ findmnt /mnt/rosetta
TARGET       SOURCE  FSTYPE   OPTIONS
/mnt/rosetta rosetta virtiofs rw,relatime,seclabel
```

```sh
ori@myfedora:~/work$ ls -l /mnt/rosetta
total 436
-rwxr-xr-x. 1 ori ori 1660888 May 22 08:09 rosetta
-rwxr-xr-x. 1 ori ori  298680 May 22 08:09 rosettad
```

## straceしてみる

```sh
ori@myfedora:~/work$ strace /mnt/rosetta/rosetta
execve("/mnt/rosetta/rosetta", ["/mnt/rosetta/rosetta"], 0xfffff242ddf0 /* 31 vars */) = 0
openat(AT_FDCWD, "/proc/self/exe", O_RDONLY) = 3
ioctl(3, _IOC(_IOC_READ, 0x61, 0x22, 0x45), 0xffffc32340e8) = 1
close(3)                                = 0
gettid()                                = 29760
awrite(2, "Usage: rosetta <x86_64 ELF to ru"..., 165Usage: rosetta <x86_64 ELF to run>

Optional environment variables:
ROSETTA_DEBUGSERVER_PORT    wait for a debugger connection on given port

version: Rosetta-318.8
) = 165
exit(1)                                 = ?
+++ exited with 1 +++
```

rosettaを実行すると、いきなり `/proc/self/exe` に対して謎のioctl(2)を発行していることがわかります。

ところでioctl(2) の `type = 0x61` (`'a'`) はどのようなデバイスなのでしょうか。[Linuxカーネルのドキュメント](https://docs.kernel.org/userspace-api/ioctl/ioctl-number.html)によると、ATMやSonetと、Intel QuickAssist Technologyのカーネルモジュールで使われている番号のようです。Fedora 40のカーネルソースで雑にgrepすると、他にHDMI CECでも使われているようでした。

```sh
ori@myfedora:~/rpmbuild/BUILD/kernel-6.10.6/linux-6.10.6-200.fc40.aarch64$ grep -lEr "_IO.*\(('a'|0x61)" include drivers
include/linux/efi.h
include/uapi/linux/atm_eni.h
include/uapi/linux/atm_he.h
include/uapi/linux/atm_idt77105.h
include/uapi/linux/atm_nicstar.h
include/uapi/linux/atm_tcp.h
include/uapi/linux/atm_zatm.h
include/uapi/linux/atmarp.h
include/uapi/linux/atmclip.h
include/uapi/linux/atmdev.h
include/uapi/linux/atmlec.h
include/uapi/linux/atmmpc.h
include/uapi/linux/atmsvc.h
include/uapi/linux/cec.h
include/uapi/linux/sonet.h
```

## ioctl(2)の発行

straceの出力と上記ブログ記事に載っているコードを参考に、下記のようなコードを作成します。

- a.c
```c
/* usage: `./a.out PATH_TO_ROSETTA_BINARY` */
/* see also: https://threedots.ovh/blog/2022/06/quick-look-at-rosetta-on-linux/ */

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/ioctl.h>

#define KEY_SIZE 0x45

int main(int argc, char *argv[])
{
	if (argc != 2) {
		fprintf(stderr, "usage: %s path\n", argv[0]);
		exit(1);
	}

	int fd = openat(AT_FDCWD, argv[1], 0);
	if (fd < 0) {
		perror("openat");
		exit(1);
	}

	char key[KEY_SIZE];
	int result = ioctl(fd, _IOC(_IOC_READ, 0x61, 0x22, KEY_SIZE), key);
	if (result < 0) {
		perror("ioctl");
		exit(1);
	}

	printf("%s\n", key);

	exit(0);
}
```

```sh
ori@myfedora:~/work$ gcc a.c
```

実行すると...謎の文字列が返ってきました！

```sh
ori@myfedora:~/work$ ./a.out /mnt/rosetta/rosetta
Our hard work
by these words guarded
please don't steal
© Apple Inc
```

## 余談

実際にx86_64なバイナリを指定してrosettaを実行すると、実は `/proc/self/exe` に対してioctl(2)を2回発行していることがわかります。

```sh
ori@myfedora:~/work$ strace /mnt/rosetta/rosetta ./peco-x86_64 a.c 2>&1 | head -n 9
execve("/mnt/rosetta/rosetta", ["/mnt/rosetta/rosetta", "./peco-x86_64", "a.c"], 0xffffe006e930 /* 31 vars */) = 0
openat(AT_FDCWD, "/proc/self/exe", O_RDONLY) = 3
ioctl(3, _IOC(_IOC_READ, 0x61, 0x22, 0x45), 0xfffff3eb1708) = 1
close(3)                                = 0
gettid()                                = 32591
getpid()                                = 32591
openat(AT_FDCWD, "/proc/self/exe", O_RDONLY) = 3
ioctl(3, _IOC(_IOC_READ, 0x61, 0x23, 0x80), 0xfffff3eb1708) = 1
close(3)
```

2回目のioctl(2)の出力は、`/run/rosettad/rosetta.sock` を返していました。

# libkrunとRosetta

冒頭でも書いたとおり、libkrunはVirtualization.frameworkを使わないVMMにもかかわらず、[Rosettaサポートを導入しました](https://github.com/containers/libkrun/pull/88)。すごい。
...と思いきや、いろいろあって、いつのまにかその[コードは消えて](https://github.com/containers/libkrun/pull/176/commits/6c2b9289b6a39826c9505f2fad5b04cc83982165)使えなくなっています。残念。

## libkrunでrosettaバイナリに対してioctlを発行したときの動き

さて、libkrunがこのRosettaのチェックをどうかいくぐっているかというと... 結論から言うと、仮想マシンがrosettaバイナリに対して上記のioctl(2)を発行したときに呼ばれるVMM内のハンドラで、`${HOME}/.krunvm-rosetta` の内容を返す実装になっています。つまりこのファイルに、上で確認した文字列を書いておけば...というわけです。

では実装を確認していきます。以下では、revert前の、commit id [af437664855a6dc27954b31f8fef2c27df37748b](https://github.com/containers/libkrun/tree/af437664855a6dc27954b31f8fef2c27df37748b) を参照します。

virtiofsを表すstruct `PassthroughFs` に、[`rosetta_data` フィールド](https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L503)があります (流用元の[crosvmのPassthroughFs](https://github.com/google/crosvm/blob/0d918fc6fde7879a4677f91c5692b2d7a6f5175a/devices/src/virtio/fs/passthrough.rs#L671)にはこのフィールドはありません)。

- https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L480-L505
```rust
/// A file system that simply "passes through" all requests it receives to the underlying file
/// system. To keep the implementation simple it servers the contents of its root directory. Users
/// that wish to serve only a specific directory should set up the environment so that that
/// directory ends up as the root of the file system process. One way to accomplish this is via a
/// combination of mount namespaces and the pivot_root system call.
pub struct PassthroughFs {
    inodes: RwLock<MultikeyBTreeMap<Inode, InodeAltKey, Arc<InodeData>>>,
    next_inode: AtomicU64,
    init_inode: u64,
    path_cache: Mutex<BTreeMap<Inode, Vec<String>>>,
    file_cache: Mutex<LruCache<Inode, Arc<File>>>,
    pinned_files: Mutex<BTreeMap<Inode, Arc<File>>>,

    handles: RwLock<BTreeMap<Handle, Arc<HandleData>>>,
    next_handle: AtomicU64,
    init_handle: u64,

    host_volumes: RwLock<HashMap<String, Inode>>,

    // Whether writeback caching is enabled for this directory. This will only be true when
    // `cfg.writeback` is true and `init` was called with `FsOptions::WRITEBACK_CACHE`.
    writeback: AtomicBool,

    rosetta_data: Option<Vec<u8>>,
    cfg: Config,
}
```

`rosetta_data` フィールドの初期化は、[コンストラクタの中で `read_rosetta_data()` を呼び出す](https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L542)ことで行います。この中で、[`${HOME}/.krunvm-rosetta` を読んで](https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L510)その内容を `rosetta_data` フィールドに格納しています。

- https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L36
```rust
const KRUNVM_ROSETTA_FILE: &str = ".krunvm-rosetta";
```

- https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L507-L523
```rust
fn read_rosetta_data() -> io::Result<Vec<u8>> {
    let home = std::env::var("HOME")
        .map_err(|_| linux_error(io::Error::from_raw_os_error(libc::EINVAL)))?;
    let path = format!("{}/{}", home, KRUNVM_ROSETTA_FILE);
    let mut file =
        File::open(path).map_err(|_| linux_error(io::Error::from_raw_os_error(libc::EINVAL)))?;

    let metadata = file
        .metadata()
        .map_err(|_| linux_error(io::Error::from_raw_os_error(libc::EINVAL)))?;

    let mut data = vec![0u8; metadata.len() as usize];
    file.read_exact(&mut data)
        .map_err(|_| linux_error(io::Error::from_raw_os_error(libc::EINVAL)))?;

    Ok(data)
}
```

libkrunのioctl(2)のハンドラの中で、[ioctlのtypeが `0x61` であれば](https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L2234) `rosetta_data` フィールドの値を返しています。

- https://github.com/containers/libkrun/blob/af437664855a6dc27954b31f8fef2c27df37748b/src/devices/src/virtio/fs/macos/passthrough.rs#L2223-L2247
```rust
    fn ioctl(
        &self,
        _ctx: Context,
        _inode: Self::Inode,
        _ohandle: Self::Handle,
        _flags: u32,
        cmd: u32,
        _arg: u64,
        _in_size: u32,
        out_size: u32,
    ) -> io::Result<Vec<u8>> {
        if cmd == IOCTL_ROSETTA {
            // This is based on the information found in this article:
            // https://threedots.ovh/blog/2022/06/quick-look-at-rosetta-on-linux/

            if let Some(data) = &self.rosetta_data {
                if data.len() == out_size as usize {
                    return Ok(data.clone());
                }
            }
            Err(linux_error(io::Error::from_raw_os_error(libc::ENOSYS)))
        } else {
            Err(linux_error(io::Error::from_raw_os_error(libc::ENOSYS)))
        }
    }
```

というわけで、あらかじめ `${HOME}/.krunvm-rosetta` を作って、しかるべき文字列を書いておくことで、libkrun上のLinux仮想マシンで、Rosettaを使ってx86_64バイナリを実行できるようになります。

## 余談

libkrunはライブラリ形式のVMMです。libkrunを使ってコマンドラインから仮想マシンを実行するツールとして、[krunvm](https://github.com/containers/krunvm)があります。
krunvmのコードには、「rosettaを実行するには`${HOME}/.krunvm-rosetta` にしかるべき文字列をあらかじめ書いておく必要がある」旨の残骸が今でも残っています。

- https://github.com/containers/krunvm/blob/765614c7837336df9afea117763c5a3fde36f245/src/commands/create.rs#L98-L113
```rust
            let path = format!("{}/{}", home, KRUNVM_ROSETTA_FILE);
            if !Path::new(&path).is_file() {
                println!(
                    "
To use Rosetta for Linux you need to create the file...

{}

...with the contents that the \"rosetta\" binary expects to be served from
its specific ioctl.

For more information, please refer to this post:
https://threedots.ovh/blog/2022/06/quick-look-at-rosetta-on-linux/
",
                    path
                );
```

[^1]: https://developer.apple.com/documentation/virtualization/running_intel_binaries_in_linux_vms_with_rosetta
