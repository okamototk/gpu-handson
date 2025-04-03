# GPUハンズオン・ハッカソン

このドキュメントはGPUハンズオン・ハッカソンのための資料です。

1. Mac(Apple Silliconで参加)
  * k8sでGPUを利用することができます。下記のURLをご覧になり進めてください。
  * https://github.com/okamototk/k8s-mac
2. Windows Intel Core i7 11thより新しいプロセッサ
  * k8sでの動作方法は確立されていません。Ollamaを動作させる時にGPUを利用することができます。
  * 下記の連載のGPU利用の節にipex-llmの利用方法をご覧ください
  * TODO:リンク
3. Windows + GeForce
  * Ollamaは普通に使うことができます。1
  * k8sで利用する場合は、下記の環境が必要となります。
    * WSL2
    * WSL CUDA Driver: https://developer.nvidia.com/cuda/wsl
    上記をインストールした上でk8s, [NVIDIA Device Driver](https://github.com/NVIDIA/k8s-device-plugin)プラグインなどを利用すると動作する可能性があります。
4. Linux + GeForce
  * Ollamaは普通に使うことができます。
  * k8sで利用する場合、NVIDIA Driver、NVIDIA Container Runtimeを手動でインストールしてGPU Operatorをインストールすれば、動作するかもしれません。
  * 
