---
title: "OpenShiftのFIPS準拠モード"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RHEL", "OpenShift", "Kubernetes", "Go", "FIPS"]
published: true
---

# はじめに

本記事では、Red Hat製品、特にRHELとOpenShiftをFIPS準拠モードで使用するためにはどうするか、どのような仕組みによってFIPS準拠モードで稼働するか、について調べた内容を書きました。

KubernetesをFIPS準拠させるのはなかなか大変で、実際Red Hatも「OpenShiftはFIPSに対応していたつもりが実はできてませんでしたすみません」という[CVE](https://access.redhat.com/security/cve/CVE-2023-3089)を発行したりしています (CVEの技術的背景等はこちらに[解説](https://access.redhat.com/security/vulnerabilities/RHSB-2023-001)があります)。

なお、本ドキュメントでは便宜上、FIPSの承認ステータスがFIPS validatedなものもModules in Progressなものもまとめて「RHEL(もしくはOpenShift)はFIPS準拠している」「FIPS準拠した暗号化モジュールを使って」のような表現をしていますのでご注意ください。正確にはざっくりRHEL 8系はほぼ準拠していますがRHEL 9系はまだ申請中の状態です。

# FIPSとは

FIPS (Federal Information Processing Standards) とは、軍事以外の用途で利用する情報通信機器が満たすべき技術標準について、アメリカ連邦政府機関定めた規格です。FIPSの個々の規格には通し番号がついています。本ドキュメントで言及するのはFIPS-140という暗号化モジュールが満たすべき基準です。最新版は2019年に発行された[FIPS 140-3](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.140-3.pdf)です。

FIPS 140の対象は暗号化モジュールです。RHELの場合、具体的にはOpenSSL, GnuTLS, NSS, libgcrypt, Kernel Crypto APIが対象となります。

Red Hat製品のFIPS含めた各種セキュリティへの準拠状況は [Compliance Activities and Government Standards](https://access.redhat.com/articles/compliance_activities_and_gov_standards) にまとまっています。RHELの各マイナーリリースおよび暗号化モジュールごとの申請状況についてもこのページに詳細が載っています。

「申請状況」と書きましたが、正確にはNIST CMVP (Cryptographic Module Validation Program) プロセスといって、以下のステージからなります。

- Implementation
- Under Test
- Modules in Process
- Validated

具体的にどの暗号化モジュールがCMVPプロセスのどのステージにいるかは、NISTの [Modules in Process Listページ](https://csrc.nist.gov/projects/cryptographic-module-validation-program/modules-in-process/modules-in-process-list) で確認することができます。`Red Hat` でページ内検索してみてください。

FIPS 140のバージョンとRed Hat製品の関係は、[FIPS 140 Lifecycle Support Statement](https://access.redhat.com/articles/7042272) のページに記載されています。ざっくりまとめると、

- FIPS 140-2

  - RHEL 8.6、およびそれを使用するOpenShift 4.11および4.12はFIPS 140-2のValidatedなOpenSSL暗号化モジュールを使用します。
  - 2025年半ばまでに、FIPS 140-3への移行パスおよび新たなRHELリリースを提供する予定 (OpenShift 4.12は2026年までサポート)

- FIPS 140-3

  - RHEL 9.2 (およびRHEL 9.2のOpenSSLを使うOpenShift 4.14) のOpenSSL暗号化モジュールについて、2023年中にFIPS140-3の承認を申請する予定 (実際申請済み)。OpenShift 4.12のーザーが移行できるよう、申請は2025年Q4までに承認される予定。
  - FIPS承認されたOpenShift 4.14を3年あいだ使用できること、およびその後1年あいだの次バージョンへの移行期間を設ける予定

という状況です。

# RHELをFIPS準拠モードで動かす

RHELをFIPS準拠させるためには、インストール時に「FIPS準拠モード」でインストールする必要があります。具体的には、起動オプションに `fips=1` を追加してインストーラを起動します。

![](/images/rhel9-fips.png)

通常のOSインストールを実施した後、FIPS準拠モードに変更するコマンドも用意されています (`fips-mode-setup --enable`)。しかし、作成済みの鍵等が全てFIPSの基準を満たすよう再作成されるわけではないなど、このコマンドで完全にFIPS準拠できることが保証されているわけではありません。確実にFIPS準拠にするためには、OSインストール時に指定する必要があります。

RHELをFIPS準拠モードにすると、

- カーネル起動オプションに `fips=1` が追加される
  - 起動後、`/proc/sys/crypto/fips_enabled` の値が `1` になる (多くのユーザーランドがFIPS準拠モードで起動したかどうかの判断にこのsysctl値を使用する)
- システム全体の暗号化ポリシーがFIPS準拠になる (`update-crypto-policy --set FIPS` を実行した状態)
- dracutモジュールとして `fips` を追加したinitramfsが再作成される
  - FIPS準拠した暗号化カーネルモジュールのみをロードする
  - 暗号化アルゴリズムが想定したものかどうかセルフテストする

等の状態になります。

# RHELにおけるGoアプリケーションのFIPS対応

OpenShift/Kubernetesは主にGo言語で書かれています。Goは言語標準の暗号化ルーチンを持っており、通常はそれを使用します。一方、OpenShiftはRHELのFIPS 140準拠した暗号化モジュール (具体的にはOpenSSL) を使用するようにビルドされています。これを実現するために、Go言語のツールチェインに対して次のような変更を加えています。

- Goのupstreamのdev.boringcryptoブランチの実装を流用してOpenSSLを呼び出す
- 実行時にFIPS準拠モードかどうかを検出して動きを変える
  - FIPS準拠モードの場合: OpenSSLの暗号化ルーチンを使用する
    - ランタイムがdl_open(3)でlibcrypto.soを読み込む
  - FIPS準拠モードではない(通常の)場合: Go言語標準の暗号化ルーチンを使用する
- 環境変数 `GOLANG_FIPS=1` を設定した場合、FIPS準拠モードかどうかによらずOpenSSLを使用する
- アプリケーションのビルド時に `-tags no_openssl` オプションを指定するとOpenSSLを使用しなくなる

また、FIPS準拠のためにOpenSSLの暗号化モジュールを使用するには、`CGO_ENABLED=1` でアプリケーションをビルドする必要があります。

## 背景

Go言語標準の暗号化ルーチンは、[FIPS対応するつもりはない](https://github.com/golang/go/issues/11658#issuecomment-120448974)そうです。

一方、HeartBleedの脆弱性が発見された頃にGoogleがOpenSSLからフォークした[BoringSSL](https://boringssl.googlesource.com/boringssl)のサブセットである[BoringCrypto](https://go.dev/src/crypto/internal/boring/README)は[FIPS 140-2 Validated](https://boringssl.googlesource.com/boringssl/+/master/crypto/fipsmodule/FIPS.md)です。

Goは、暗号化ルーチンをGo標準ライブラリではなくBoringCryptoに切り替える[dev.boringcryptoブランチ](https://github.com/golang/go/tree/dev.boringcrypto)を持っていました^[dev.boringcryptoブランチは2022年5月に[masterにマージされた](https://github.com/golang/go/commit/f771edd7f92a47c276d65fbd9619e16a786c6746)ようです]。

Red HatのGoツールチェインは、このブランチをフォークして、BoringSSLではなくOpenSSLの暗号化モジュールを呼び出すようにしました。OpenShiftをFIPS準拠させるにあたって、OpenShiftとOS(RHEL)の暗号化モジュールを統一できることは、申請にかかる工数や時間の面でメリットになると考えたのだと思われます。

# OpenShiftのFIPS準拠

OpenShiftをFIPS準拠モードにするには、install-config.yamlに `fips: true` を追記してクラスターを構築します。

FIPS準拠モードのOpenShiftでは、暗号化モジュールとしてRHELのFIPS承認済みのコンポーネントを使用します。
具体的には、RHEL CoreOSが「FIPS準拠モードのRHEL」と同じ状態、つまり `fips=1` オプションがついた状態でカーネルが起動し、crypto policyが `FIPS` に設定され、dracutモジュールとしてfipsが有効になった状態で動いています^[細かいですが、CoreOSの場合はFIPS準拠モードかどうかによらずinitramfsに最初からfipsモジュールが組み込まれており、FIPS準拠モードで起動した場合のみfipsモジュールを実行するような実装になっています]。

kubeadm等で使用するKubernetesのバイナリはCGO_ENABLED=0でシングルバイナリとしてビルドされているようです (私が見た範囲では)。
一方、OpenShiftはCGO_ENABLED=1でビルドされています。

kube-apiserverのバイナリを実際に調べます。まず、kube-apiserverコンテナのイメージIDを調べます。

```
$ oc -n openshift-kube-apiserver get pod kube-apiserver-ip-10-0-0-254.us-east-2.compute.internal -o json | jq -r '.spec.containers[] | select(.name == "kube-apiserver") | .image'
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:dbb6ec887da8c17920f8ceac12c9c10c77f9c77519f3c61fb4c3154d8424db32
```

oc debug nodeでcontrol planeノードに入り、podman image mountを使ってkube-apiserverのコンテナイメージをホストOS上にマウントします。

```
$ oc debug node/ip-10-0-0-254.us-east-2.compute.internal
Starting pod/ip-10-0-0-254us-east-2computeinternal-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.254
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
```

```
# podman image mount quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:dbb6ec887da8c17920f8ceac12c9c10c77f9c77519f3c61fb4c3154d8424db32
/var/lib/containers/storage/overlay/ed23a764f024a9f64eccdcb513b90d9a24cd8459b7f88407138c5cfacd304ac3/merged
```

fileやlddを使うと、kube-apiserverのバイナリがダイナミックリンクされていることがわかります。

```
# file /var/lib/containers/storage/overlay/ed23a764f024a9f64eccdcb513b90d9a24cd8459b7f88407138c5cfacd304ac3/merged/usr/bin/kube-apiserver
/var/lib/containers/storage/overlay/ed23a764f024a9f64eccdcb513b90d9a24cd8459b7f88407138c5cfacd304ac3/merged/usr/bin/kube-apiserver: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4f82cbaaaea28ab9c195123716cdbba1d3b92790, for GNU/Linux 3.2.0, stripped
```

```
# ldd /var/lib/containers/storage/overlay/ed23a764f024a9f64eccdcb513b90d9a24cd8459b7f88407138c5cfacd304ac3/merged/usr/bin/kube-apiserver
        linux-vdso.so.1 (0x00007ffff4930000)
        libresolv.so.2 => /lib64/libresolv.so.2 (0x00007f390a06f000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f3909e00000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f390a08a000)
```

kube-apiserverのプロセス空間にマッピングされているファイルを確認すると、lddの出力にはなかった `libcrypto.so` や `/usr/lib64/ossl-modules/fips.so` がマッピングされていることがわかります。`libcrypto.so` はGoのランタイムによって、`fips.so` はOpenSSLのPrividerの仕組みによってそれぞれdl_open(3)でプロセス実行中に読み込まれたものです^[RHEL 9はOpenSSL 3系なので、FIPS対応時はfips.soをdl_open(3)しますが、RHEL 8はOpenSSL 1系なので、ファイルのマッピングの見え方は異なります (OpenSSL 1系にはfips.soは存在しないため、libcrypto.soのみdl_open(3)します)]^[libcrypto.soに関しては、FIPS準拠モードかどうかに関わらず、Goランタイムがdo_open(3)します。FIPS準拠モードで稼働しているかどうかを実行時に判断するために、libcrypto.soの[関数](https://www.openssl.org/docs/man3.0/man3/EVP_default_properties_is_fips_enabled.html)を呼び出すためです]。

```
# ls -l /proc/$(pidof kube-apiserver)/map_files
total 0
lr--------. 1 root root 64 May 28 00:09 400000-406000 -> /usr/bin/kube-apiserver
lr--------. 1 root root 64 May 28 00:09 406000-42b8000 -> /usr/bin/kube-apiserver
lr--------. 1 root root 64 May 28 00:09 42b8000-8561000 -> /usr/bin/kube-apiserver
lr--------. 1 root root 64 May 28 00:09 7f616f3cd000-7f616f47a000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f616f47a000-7f616f6d6000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f616f6d6000-7f616f7a2000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f616f7a2000-7f616f7a3000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f616f7a3000-7f616f7f9000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f616f7f9000-7f616f7fc000 -> /usr/lib64/libcrypto.so.3.0.7
lr--------. 1 root root 64 May 28 00:09 7f617c049000-7f617c05d000 -> /usr/lib64/ossl-modules/fips.so
lr--------. 1 root root 64 May 28 00:09 7f617c05d000-7f617c151000 -> /usr/lib64/ossl-modules/fips.so
lr--------. 1 root root 64 May 28 00:09 7f617c151000-7f617c179000 -> /usr/lib64/ossl-modules/fips.so
lr--------. 1 root root 64 May 28 00:09 7f617c179000-7f617c18b000 -> /usr/lib64/ossl-modules/fips.so
lr--------. 1 root root 64 May 28 00:09 7f617c18b000-7f617c18c000 -> /usr/lib64/ossl-modules/fips.so
lr--------. 1 root root 64 May 28 00:09 7f617c18c000-7f617c18f000 -> /usr/lib64/libz.so.1.2.11
lr--------. 1 root root 64 May 28 00:09 7f617c18f000-7f617c19d000 -> /usr/lib64/libz.so.1.2.11
lr--------. 1 root root 64 May 28 00:09 7f617c19d000-7f617c1a3000 -> /usr/lib64/libz.so.1.2.11
lr--------. 1 root root 64 May 28 00:09 7f617c1a3000-7f617c1a4000 -> /usr/lib64/libz.so.1.2.11
lr--------. 1 root root 64 May 28 00:09 7f617c1a4000-7f617c1a5000 -> /usr/lib64/libz.so.1.2.11
lr--------. 1 root root 64 May 28 00:09 7f61a56e2000-7f61a570a000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a570a000-7f61a587f000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a587f000-7f61a58d7000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a58d7000-7f61a58d8000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a58d8000-7f61a58dc000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a58dc000-7f61a58de000 -> /usr/lib64/libc.so.6
lr--------. 1 root root 64 May 28 00:09 7f61a58eb000-7f61a58ef000 -> /usr/lib64/libresolv.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a58ef000-7f61a58f8000 -> /usr/lib64/libresolv.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a58f8000-7f61a58fb000 -> /usr/lib64/libresolv.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a58fb000-7f61a58fc000 -> /usr/lib64/libresolv.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a58fc000-7f61a58fd000 -> /usr/lib64/libresolv.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a5904000-7f61a5906000 -> /usr/lib64/ld-linux-x86-64.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a5906000-7f61a592c000 -> /usr/lib64/ld-linux-x86-64.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a592c000-7f61a5937000 -> /usr/lib64/ld-linux-x86-64.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a5938000-7f61a593a000 -> /usr/lib64/ld-linux-x86-64.so.2
lr--------. 1 root root 64 May 28 00:09 7f61a593a000-7f61a593c000 -> /usr/lib64/ld-linux-x86-64.so.2
lr--------. 1 root root 64 May 28 00:09 8561000-8562000 -> /usr/bin/kube-apiserver
lr--------. 1 root root 64 May 28 00:09 8562000-866f000 -> /usr/bin/kube-apiserver
```

# 参考文献

本文中ではリンクを載せていませんが参考になったページを下記に記します。

- [FIPS compliant crypto in golang](https://kupczynski.info/posts/fips-golang/)
- [RH article on FIPS-compliant Go](https://www.reddit.com/r/golang/comments/vbkm49/rh_article_on_fipscompliant_go/) (よく見たら回答しているのがsmarterclayton氏だった)
- [Navigating FIPS Compliance for Go Applications: Libraries, Integration, and Security](https://medium.com/cyberark-engineering/navigating-fips-compliance-for-go-applications-libraries-integration-and-security-42ac87eec40b)