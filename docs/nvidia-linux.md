# NVIDIA GPU + Linux11でKubernetesでGPUを利用する方法

基本的にNVIDIAのエンタープライズのGPU(A100,H100,H200,GBXXX)は、GPU OperatorがドライバのインストールからDevice Pluginのインストールまで必要な処理はすべてやってくれます。
GPU Operatorは基本的にGeForceなどのコンシューマ向けのGPUに対応していませんが、手動でセットアップすることにより1Kubernetesから利用することができます。
ここでは、手動でGPU Operatorを利用してセットアップを行い、GeForceなどGPU OperatorがサポートしていないGPUでKubernetesを利用する手順を紹介します。

## 事前準備
DockerやContainerd上で動作するKubernetesをセットアップしてください。Rancher Desktopは現状サポートしていないようなので注意してください。
OSはNVIDIAのBCM(OpenShiftみたいなもの)がUbuntuを利用しているので、Ubuntuがいいと思いますが、RedHat系でも多分動くと思います。

## Kubernetesのセットアップ
### 1. NVIDIAドライバのインストール
NVIDIAドライバをインストールします。お使いのGPUに応じたドライバをインストールしてください。

```
$ sudo apt install nvidia-driver-550  # Use a compatible driver version
$ sudo reboot
```

> [!NOTE]  
> WSLの場合、ホストのWindows用のドライバの他、CUDA Driver for WSLをインストールする必要があります。
> 詳細は[Enable NVIDIA CUDA on WSL](https://learn.microsoft.com/en-us/windows/ai/directml/gpu-cuda-in-wsl)をご覧ください。

カーネルモジュールの確認

```
$ lsmod |grep nvidia
nvidia_uvm           5025792  4
nvidia_drm            122880  0
nvidia_modeset       1507328  1 nvidia_drm
nvidia               8835072  7 nvidia_uvm,nvidia_modeset
video                  73728  1 nvidia_modeset
ecc                    45056  1 nvidia
```

入ってなければロードする。

```
$ sudo modprobe nvidia
```

失敗した場合、GPUがドライバに対応していない可能性があるので、対応するNVIDIA Driverをインストール。


### 2. NVIDIA Container Toolkitのインストール
ここの手順に従う

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

```
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

#### Dockerの場合
```
sudo su -
nvidia-ctk runtime configure
systemctl restart docker
```

#### Containerd(k3s)の場合

```
sudo su - 
nvidia-ctk runtime configure --runtime=containerd --config /var/lib/rancher/k3s/agent/etc/containerd/config.toml
systemctl restart k3s
```

config.tomlのパスは環境に応じて変更すること。

### 3 GPUノードにラベル追加

```
$ kubectl get nodes
NAME           STATUS   ROLES                  AGE   VERSION
ip-10-0-16-4   Ready    control-plane,master   21h   v1.31.6+k3s1
```

GPUを利用するノードでラベルを追加

```
kubectl label node ip-10-0-16-4 nvidia.com/gpu.present=true
```

### 3. GPU Operatorのインストール

```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia  # Use the OSS-compatible repo
helm repo update
```

#### PSAが有効な場合下記のてじゅんでインストール
```
kubectl label --overwrite ns gpu-operator pod-security.kubernetes.io/enforce=privileged 
helm install --wait --generate-name \
   -n gpu-operator --create-namespace \
   nvidia/gpu-operator \
  --set driver.enabled=false \
  --set toolkit.enabled=false \
  --set devicePlugin.enabled=true
```

### 4. 動作確認

Operator Podが動作しているか確認

```
kubectl get pods -n gpu-operator
```

ノードにnvidia.com/gpuが追加されているか確認。

```
kubectl describe node
...
Capacity:
  cpu:                4
  ephemeral-storage:  506771172Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16167136Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  492986995735
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16167136Ki
  nvidia.com/gpu:     1
  pods:               110
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                515m (12%)  500m (12%)
  memory             560Mi (3%)  6152Mi (38%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  nvidia.com/gpu     0           0

```

### 5. Podの動作確認

#### gpu-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: gpu-operator-test
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      # https://catalog.ngc.nvidia.com/orgs/nvidia/teams/k8s/containers/cuda-sample
      #      image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
      image: "nvcr.io/nvidia/cuda:12.1.0-base-ubuntu22.04"
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      resources:
        limits:
          nvidia.com/gpu: 1
```

```
kubectl apply -f gpu-pod.yaml
```

```
kubectl exec -it gpu-operator-test -- bash

apt-get update
apt-get install nvidia-utils-550 # nvidia-utilがPodに入っていないのでインストール
nvidia-smi
```

### GeForceの制限事項
＊ MIGによるGPUの論理分割は利用できない。1
* DCや商用として利用するのは禁止されている。非商用の目的のために利用すること。
