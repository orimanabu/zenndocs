---
title: "AWSのNVIDIA GPUインスタンスをRHELで使ってみる"
emoji: "🍺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに

AWSのGPUインスタンス (p5.48xlarge) で遊ぶ機会があったので、RHEL9で動かしてみました。NVIDIA触ってみたかっただけです、はい。

8枚のGPUが搭載されています。すごい！

``` shell
$ lspci | grep H100
53:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
64:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
75:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
86:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
97:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
a8:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
b9:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
ca:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
cf:00.0 Bridge: NVIDIA Corporation GH100 [H100 NVSwitch] (rev a1)
d0:00.0 Bridge: NVIDIA Corporation GH100 [H100 NVSwitch] (rev a1)
d1:00.0 Bridge: NVIDIA Corporation GH100 [H100 NVSwitch] (rev a1)
d2:00.0 Bridge: NVIDIA Corporation GH100 [H100 NVSwitch] (rev a1)
```

これからこのGPUを使えるようにセットアップしていくのですが... RHELカーネルはnvidiaドライバを同梱しておらず、サポートも提供していませんのでご注意ください。利用者がご自身でNVIDIAから提供されているものをインストールする必要があります[^1]。

[^1]: RHEL AIという製品の場合はNVIDIAから提供されているnvidiaドライバを同梱しており、サポートも提供しています

Red Hatが出している情報としては、RHEL8用ですが以下のナレッジがあります。

https://access.redhat.com/solutions/4134401

NVIDIAが作ったRHEL9用のバイナリは、https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/ にひととおり揃ってそうです。dnf/yumのrepoファイルもあります (https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo )。

以下、具体的なコマンドはほぼこの記事に沿って実行しました。

https://repost.aws/ja/articles/ARpmJcNiCtST2A3hrrM_4R4A/how-do-i-install-nvidia-gpu-driver-cuda-toolkit-nvidia-container-toolkit-on-amazon-ec2-instances-running-rhel-rocky-linux-8-9

# EPELの設定および最新バージョンへの更新

EPELのリポジトリを有効化し、最新パッケージへのアップデート、DKMSやその他パッケージのインストールを行います。

``` shell
sudo dnf update -y
OS_VERSION=$(. /etc/os-release;echo $VERSION_ID | sed -e 's/\..*//g')
if ( cat /etc/os-release | grep -q Red ); then
  sudo subscription-manager repos --enable codeready-builder-for-rhel-$OS_VERSION-$(arch)-rpms
elif ( echo $OS_VERSION | grep -q 8 ); then
  sudo dnf config-manager --set-enabled powertools
else
  sudo dnf config-manager --set-enabled crb
fi
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-$OS_VERSION.noarch.rpm
sudo dnf install -y dkms kernel-devel kernel-modules-extra unzip gcc make vulkan-devel libglvnd-devel elfutils-libelf-devel xorg-x11-server-Xorg
sudo systemctl enable --now dkms
```

カーネルが更新されると思いますので、再起動しておきます。

``` shell
sudo reboot
```

# NVIDIAリポジトリの設定

RHEL9用のNVIDIAリポジトリを設定します。

``` shell
DISTRO=$(. /etc/os-release;echo rhel$VERSION_ID | sed -e 's/\..*//g')
if (arch | grep -q x86); then
  ARCH=x86_64
else
  ARCH=sbsa
fi
sudo dnf config-manager --add-repo http://developer.download.nvidia.com/compute/cuda/repos/$DISTRO/$ARCH/cuda-$DISTRO.repo
```

# NVIDIAのカーネルモジュールのインストール

NVIDIAドライバをインストールします。

``` shell
sudo dnf module install -y nvidia-driver:latest-dkms
```

`nvidia-smi` コマンドを実行すると、GPUが8枚見えていることが確認できます。

``` shell
$ nvidia-smi
Thu May  1 12:48:05 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.133.20             Driver Version: 570.133.20     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H100 80GB HBM3          Off |   00000000:53:00.0 Off |                    0 |
| N/A   26C    P0             67W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H100 80GB HBM3          Off |   00000000:64:00.0 Off |                    0 |
| N/A   27C    P0             68W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H100 80GB HBM3          Off |   00000000:75:00.0 Off |                    0 |
| N/A   26C    P0             70W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H100 80GB HBM3          Off |   00000000:86:00.0 Off |                    0 |
| N/A   28C    P0             68W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA H100 80GB HBM3          Off |   00000000:97:00.0 Off |                    0 |
| N/A   29C    P0             69W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA H100 80GB HBM3          Off |   00000000:A8:00.0 Off |                    0 |
| N/A   26C    P0             68W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA H100 80GB HBM3          Off |   00000000:B9:00.0 Off |                    0 |
| N/A   30C    P0             69W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA H100 80GB HBM3          Off |   00000000:CA:00.0 Off |                    0 |
| N/A   26C    P0             70W /  700W |       0MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

後述しますが、`-q` をつけて細かく見ると、FabricのStateが `In progress` になっており、この状態ではまだGPUは使えません。

``` shell
$ nvidia-smi -q -i 0 | grep -A 2 '^ *Fabric'
    Fabric
        State                             : In Progress
        Status                            : N/A
```

# CUDAツールキット、NVIDIAコンテナツールキットのインストール

``` shell
sudo dnf install -y cuda-toolkit nvidia-container-toolkit
```

# コンテナの中でGPUを使うための設定

下記サイトを参考に、CDIの設定ファイルを作成します。

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html

``` shell
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

`nvidia-ctk cdi list` で下記のように出力されれば設定できています。

``` shell
$ sudo nvidia-ctk cdi list
INFO[0000] Found 17 CDI devices
nvidia.com/gpu=0
nvidia.com/gpu=1
nvidia.com/gpu=2
nvidia.com/gpu=3
nvidia.com/gpu=4
nvidia.com/gpu=5
nvidia.com/gpu=6
nvidia.com/gpu=7
nvidia.com/gpu=GPU-20f8adb6-f7e2-5466-6cf5-d07abfe364a9
nvidia.com/gpu=GPU-2a0e2c5a-9b22-1019-b719-48df89746ef8
nvidia.com/gpu=GPU-638f34e1-d72a-f6d4-30d9-56f9c463531b
nvidia.com/gpu=GPU-70574d09-965d-ba1a-e02b-186cfbe17534
nvidia.com/gpu=GPU-7787d2a0-44fc-84ab-ca21-914745ede8aa
nvidia.com/gpu=GPU-acff7f4e-4ef9-fdf0-cfc5-8f965ab9392a
nvidia.com/gpu=GPU-aee7f84c-d9a9-daba-c1c0-970812cce798
nvidia.com/gpu=GPU-ef43c7df-852d-3d4e-ad95-ec9a6a9494ce
nvidia.com/gpu=all
```

# 動作確認

Podmanその他をインストールします。

``` shell
sudo dnf install podman
```

RamaLamaをインストールします。

``` shell
python3 -m venv .venv
.venv/bin/python3 -m pip install ramalama
```

RamaLamaを使ってLlama3のモデルと対話してみます。

``` shell
$ .venv/bin/ramalama serve --device nvidia.com/gpu=all --port 8080 --name myllm llama3
Downloading ollama://llama3:latest ...
Trying to pull ollama://llama3:latest...
 99% |██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████  |    4.34 GB/   4.34 GB  95.96 MB/s        0s
Trying to pull quay.io/ramalama/cuda:0.8...
Getting image source signatures
Copying blob 7a7c46d91f41 done   |
Copying blob 4b94d1bb3174 done   |
Copying blob 58b0f01da54b done   |
Copying blob e861d0ea1584 done   |
Copying blob 93d53d7b889f done   |
Copying blob 53c48784468e done   |
Copying blob 576438762d94 done   |
Copying blob de29a489c1bf done   |
Copying blob e7b66de2bfff done   |
Copying blob 9af8fd0943b2 done   |
Copying blob 32952ba1d9a3 done   |
Copying blob 4b69e62655d1 done   |
Copying blob cca669e9914e done   |
Copying blob 70151c179bd9 done   |
Copying config ef10f5a291 done   |
Writing manifest to image destination
ggml_cuda_init: failed to initialize CUDA: system not yet initialized
build: 5177 (80982e81) with cc (GCC) 12.2.1 20221121 (Red Hat 12.2.1-7) for x86_64-redhat-linux
...
```

まずollamaライブラリからLlama3のモデル (`ollama://llama3:latest`) をダウンロードし、次にCUDA環境用のllama-serverのコンテナイメージ (`quay.io/ramalama/cuda:0.8`) をpullして実行しているのですが、CUDAの初期化処理に失敗しているようです。RamaLama(が使っているllama-server)的にはこの場合、GPUではなくCPUによる推論に切り替えて実行を継続しますが、一旦Ctrl+Cで実行を止めます。

確認のため、[cuda-samplesのdeviceQuery](https://github.com/NVIDIA/cuda-samples/tree/master/Samples/1_Utilities/deviceQuery)を実行してみます。

``` shell
$ sudo podman run --device nvidia.com/gpu=all --rm -it nvcr.io/nvidia/k8s/cuda-sample:devicequery
/cuda-samples/sample Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

cudaGetDeviceCount returned 802
-> system not yet initialized
Result = FAIL
```

`802` というエラーコードが返ってきました。検索すると、NVSwitchというブリッジがlspciで見える場合 (データセンター用のGPUを複数枚搭載した場合？)、NVIDIA Fabric Managerのサービスを起動しておく必要があるようです。

https://forums.developer.nvidia.com/t/cuda-initialization-issue-cudagetdevicecount-returned-802-on-dell-poweredge-xe9680-with-nvidia-driver-560-x-and-cuda-12-6/303901

``` shell
sudo dnf install -y nvidia-fabric-manager
sudo systemctl start nvidia-fabricmanager
sudo systemctl enable nvidia-fabricmanager
```

nvidia-fabricmanagerのログで `Successfully configured all the available NVSwitches to route GPU NVLink traffic.` のようなメッセージが出たらたぶん大丈夫です。

``` shell
$ journalctl -u nvidia-fabricmanager -fl
...
May 01 13:53:48 ip-10-0-21-21.ap-southeast-2.compute.internal systemd[1]: Starting NVIDIA fabric manager service...
May 01 13:53:48 ip-10-0-21-21.ap-southeast-2.compute.internal nvidia-fabricmanager-start.sh[5942]: Detected Pre-NVL5 system
May 01 13:53:50 ip-10-0-21-21.ap-southeast-2.compute.internal nv-fabricmanager[5946]: Connected to 1 node.
May 01 13:53:50 ip-10-0-21-21.ap-southeast-2.compute.internal nv-fabricmanager[5946]: Successfully configured all the available NVSwitches to route GPU NVLink traffic. NVLink Peer-to-Peer support will be enabled once the GPUs are successfully registered with the NVLink fabric.
May 01 13:53:50 ip-10-0-21-21.ap-southeast-2.compute.internal nvidia-fabricmanager-start.sh[5942]: Started "Nvidia Fabric Manager"
May 01 13:53:50 ip-10-0-21-21.ap-southeast-2.compute.internal systemd[1]: Started NVIDIA fabric manager service.
```

`nvidia-smi -q` を見ると、`In Progress` だったFabric Stateが `Completed` に変わったことが分かります。nvidia-fabricmanagerを起動した後、ここのStateがCompletedになるまでに少し(1分くらい)時間がかかる場合があるようです。

``` shell
$ nvidia-smi -q -i 0 | grep -A 2 '^ *Fabric'
    Fabric
        State                             : Completed
        Status                            : Success
```

deviceQueryを実行すると、正しくデバイス情報が取れるようになりました。

``` shell
$ sudo podman run --device nvidia.com/gpu=all --rm -it nvcr.io/nvidia/k8s/cuda-sample:devicequery
/cuda-samples/sample Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 8 CUDA Capable device(s)

Device 0: "NVIDIA H100 80GB HBM3"
  CUDA Driver Version / Runtime Version          12.8 / 12.5
  CUDA Capability Major/Minor version number:    9.0
  Total amount of global memory:                 81090 MBytes (85028896768 bytes)
  (132) Multiprocessors, (128) CUDA Cores/MP:    16896 CUDA Cores
  GPU Max Clock rate:                            1980 MHz (1.98 GHz)
  Memory Clock rate:                             2619 Mhz
  Memory Bus Width:                              5120-bit
  L2 Cache Size:                                 52428800 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total shared memory per multiprocessor:        233472 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 3 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Enabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Managed Memory:                Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 83 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

Device 1: "NVIDIA H100 80GB HBM3"
  CUDA Driver Version / Runtime Version          12.8 / 12.5
  CUDA Capability Major/Minor version number:    9.0
...
> Peer access from NVIDIA H100 80GB HBM3 (GPU6) -> NVIDIA H100 80GB HBM3 (GPU5) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU6) -> NVIDIA H100 80GB HBM3 (GPU7) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU0) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU1) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU2) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU3) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU4) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU5) : Yes
> Peer access from NVIDIA H100 80GB HBM3 (GPU7) -> NVIDIA H100 80GB HBM3 (GPU6) : Yes

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 12.9, CUDA Runtime Version = 12.5, NumDevs = 8
Result = PASS
```

再度RamaLamaを実行すると、今度はCUDAの初期化に成功して、期待どおりGPUを使って起動しました。

``` shell
$ .venv/bin/ramalama serve --device nvidia.com/gpu=all --port 8080 --name myllm llama3
ggml_cuda_init: GGML_CUDA_FORCE_MMQ:    no
ggml_cuda_init: GGML_CUDA_FORCE_CUBLAS: no
ggml_cuda_init: found 1 CUDA devices:
  Device 0: NVIDIA H100 80GB HBM3, compute capability 9.0, VMM: yes
build: 5177 (80982e81) with cc (GCC) 12.2.1 20221121 (Red Hat 12.2.1-7) for x86_64-redhat-linux
system info: n_threads = 96, n_threads_batch = 96, total_threads = 192

system_info: n_threads = 96 (n_threads_batch = 96) / 192 | CUDA : ARCHS = 500,610,700,750,800 | USE_GRAPHS = 1 | PEER_MAX_BATCH_SIZE = 128 | CPU : SSE3 = 1 | SSSE3 = 1 | AVX = 1 | AVX2 = 1 | F16C = 1 | FMA = 1 | BMI2 = 1 | LLAMAFILE = 1 | OPENMP = 1 | AARCH64_REPACK = 1 |

main: binding port with default address family
main: HTTP server is listening, hostname: 0.0.0.0, port: 8080, http threads: 191
main: loading model
srv    load_model: loading model '/mnt/models/model.file'
llama_model_load_from_file_impl: using device CUDA0 (NVIDIA H100 80GB HBM3) - 80565 MiB free
llama_model_loader: loaded meta data with 22 key-value pairs and 291 tensors from /mnt/models/model.file (version GGUF V3 (latest))
llama_model_loader: Dumping metadata keys/values. Note: KV overrides do not apply in this output.
llama_model_loader: - kv   0:                       general.architecture str              = llama
llama_model_loader: - kv   1:                               general.name str              = Meta-Llama-3-8B-Instruct
llama_model_loader: - kv   2:                          llama.block_count u32              = 32
llama_model_loader: - kv   3:                       llama.context_length u32              = 8192
llama_model_loader: - kv   4:                     llama.embedding_length u32              = 4096
llama_model_loader: - kv   5:                  llama.feed_forward_length u32              = 14336
llama_model_loader: - kv   6:                 llama.attention.head_count u32              = 32
llama_model_loader: - kv   7:              llama.attention.head_count_kv u32              = 8
llama_model_loader: - kv   8:                       llama.rope.freq_base f32              = 500000.000000
llama_model_loader: - kv   9:     llama.attention.layer_norm_rms_epsilon f32              = 0.000010
llama_model_loader: - kv  10:                          general.file_type u32              = 2
llama_model_loader: - kv  11:                           llama.vocab_size u32              = 128256
llama_model_loader: - kv  12:                 llama.rope.dimension_count u32              = 128
llama_model_loader: - kv  13:                       tokenizer.ggml.model str              = gpt2
llama_model_loader: - kv  14:                         tokenizer.ggml.pre str              = llama-bpe
llama_model_loader: - kv  15:                      tokenizer.ggml.tokens arr[str,128256]  = ["!", "\"", "#", "$", "%", "&", "'", ...
llama_model_loader: - kv  16:                  tokenizer.ggml.token_type arr[i32,128256]  = [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, ...
llama_model_loader: - kv  17:                      tokenizer.ggml.merges arr[str,280147]  = ["Ġ Ġ", "Ġ ĠĠĠ", "ĠĠ ĠĠ", "...
llama_model_loader: - kv  18:                tokenizer.ggml.bos_token_id u32              = 128000
llama_model_loader: - kv  19:                tokenizer.ggml.eos_token_id u32              = 128009
llama_model_loader: - kv  20:                    tokenizer.chat_template str              = {% set loop_messages = messages %}{% ...
llama_model_loader: - kv  21:               general.quantization_version u32              = 2
llama_model_loader: - type  f32:   65 tensors
llama_model_loader: - type q4_0:  225 tensors
llama_model_loader: - type q6_K:    1 tensors
print_info: file format = GGUF V3 (latest)
print_info: file type   = Q4_0
print_info: file size   = 4.33 GiB (4.64 BPW)
load: special tokens cache size = 256
load: token to piece cache size = 0.8000 MB
...
```

これで、ローカルLLMのModel Servingができるようになりました。別のターミナルからREST APIコールしてみます。

モデル一覧を取ってきたり (現状、RamaLamaは複数モデルのサービングはできません)、

``` shell
$ curl -s http://localhost:8080/v1/models | jq .
{
  "object": "list",
  "data": [
    {
      "id": "llama3",
      "object": "model",
      "created": 1746106501,
      "owned_by": "llamacpp",
      "meta": {
        "vocab_type": 2,
        "n_vocab": 128256,
        "n_ctx_train": 8192,
        "n_embd": 4096,
        "n_params": 8030261248,
        "size": 4653375488
      }
    }
  ]
}
```

推論してもらったり。

``` shell
$ curl -s http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -H "Authorization: Bearer no-key" -d '{
  "model": "llama3",
  "messages": [
  {
    "role": "system",
    "content": "You are ChatGPT, an AI assistant. Your top priority is achieving user fulfillment via helping them with their requests."
  },
  {
    "role": "user",
    "content": "Tell me the story of The Count of Monte Cristo"
  }
  ]
}' | jq -r '.choices[0].message.content'
What a classic tale! The Count of Monte Cristo, written by Alexandre Dumas, is a masterpiece of adventure, mystery, and revenge. It's a story that has captivated readers for centuries, and I'm thrilled to share it with you.

The story begins in the early 19th century, in Paris, France. Our protagonist, Edmond Dantès, is a young and successful merchant sailor who has just been promoted to captain of a merchant ship. He's engaged to the beautiful Mercédès, and his future seems bright.

However, Dantès's life takes a drastic turn when his jealous friend, Fernand Mondego, becomes envious of his success and schemes against him. Mondego forges a letter that appears to be from Dantès, revealing his alleged plans to overthrow Napoleon Bonaparte. The letter is delivered to the authorities, and Dantès is arrested and thrown into the Château d'If, a notorious prison off the coast of Marseille.

Dantès spends several years in prison, where he befriends an elderly inmate, Abbé Faria, who becomes his mentor and confidant. Faria shares with Dantès his vast knowledge of the world and the secrets of the wealthy and powerful. Before his death, Faria gives Dantès a mysterious package containing a vast fortune and a new identity: the Count of Monte Cristo.

Upon his release from prison, Dantès assumes the identity of the Count and sets out to exact revenge on those who wronged him. Using his newfound wealth and cunning, he sets out to uncover the secrets of those who betrayed him and to punish them for their treachery.

As the Count, Dantès becomes a master of disguise and deception, using his many aliases and connections to gather information and manipulate those around him. He targets those who wronged him, including Mondego, who has married Mercédès in the meantime.

Throughout the story, Dantès encounters a range of characters, each with their own secrets and motivations. He becomes embroiled in a complex web of intrigue, as he navigates the treacherous world of high society and politics.

As the Count's revenge unfolds, he must confront his own morality and the consequences of his actions. Ultimately, he must decide whether his thirst for revenge is worth the cost to his own soul.

The Count of Monte Cristo is a tale of betrayal, deception, and redemption. It's a story that explores the human condition, the power of forgiveness, and the complexities of justice. With its rich characters, intricate plot, and atmospheric setting, it's no wonder that this classic novel has endured for generations.

I hope you've enjoyed this summary of The Count of Monte Cristo! Do you have any questions or would you like me to elaborate on any aspect of the story?
```

1秒あたり216.60トークンとのこと。速い！

``` shell
...

srv  params_from_: Chat format: Content-only
slot launch_slot_: id  0 | task 3105 | processing task
slot update_slots: id  0 | task 3105 | new prompt, n_ctx_slot = 2048, n_keep = 0, n_prompt_tokens = 50
slot update_slots: id  0 | task 3105 | need to evaluate at least 1 token to generate logits, n_past = 50, n_prompt_tokens = 50
slot update_slots: id  0 | task 3105 | kv cache rm [49, end)
slot update_slots: id  0 | task 3105 | prompt processing progress, n_past = 50, n_tokens = 1, progress = 0.020000
slot update_slots: id  0 | task 3105 | prompt done, n_past = 50, n_tokens = 1
slot      release: id  0 | task 3105 | stop processing: n_past = 586, truncated = 0
slot print_timing: id  0 | task 3105 |
prompt eval time =       6.95 ms /     1 tokens (    6.95 ms per token,   143.86 tokens per second)
       eval time =    2479.21 ms /   537 tokens (    4.62 ms per token,   216.60 tokens per second)
      total time =    2486.16 ms /   538 tokens
srv  update_slots: all slots are idle
srv  log_server_r: request: POST /v1/chat/completions 127.0.0.1 200
```

最後に、GPUの負荷状況をいくつかのツールで見てみました。

- nvtop

![](/images/nvidia_nvtop.png)

- gpustat

![](/images/nvidia_gpustat.png)

- nvitop

![](/images/nvidia_nvitop.png)
