# PYTHON ENV

* workdir: /data/hex

## verl-vllm011

* reference: [verl0.6-cu128-torch2.8.0-fa2.7.4](https://github.com/volcengine/verl/blob/main/docker/verl0.6-cu128-torch2.8.0-fa2.7.4/Dockerfile.base)

```bash
# 创建 python 环境
$ uv venv --python 3.12 --seed /data/hex/.python/verl-vllm011

# $ source /data/hex/.python/verl-vllm011/bin/activate
$ uv pip install uv

$ uv pip install vllm

$ uv pip install --no-cache-dir --no-build-isolation flash_attn==2.7.4.post1

# pip install -v --disable-pip-version-check --no-cache-dir --no-build-isolation --config-settings "--build-option=--cpp_ext" --config-settings "--build-option=--cuda_ext" git+https://github.com/NVIDIA/apex.git
$ uv pip install -v --disable-pip-version-check --no-cache-dir --no-build-isolation -C="--build-option=--cpp_ext" -C="--build-option=--cuda_ext" git+https://github.com/NVIDIA/apex.git
```

## megatron-swift-torch27

* docker: 基于 nvcr.io/nvidia/pytorch:25.03 制作
    * cuda: 12.8

```bash
# 创建 python 环境
$ uv venv --python 3.12 --seed /data/hex/.python/megatron-swift-torch27
# 使用环境
# $ source /data/hex/.python/megatron-swift-torch27/bin/activate

# 先在环境里装个 uv
$ uv pip install uv

# 安装 torch 2.7
$ uv pip install torch==2.7.1 torchvision==0.22.1 torchaudio==2.7.1 --index-url https://download.pytorch.org/whl/cu128

# 直接装 ms-swift
$ uv pip install ms-swift swanlab

# 装预编译好的 flash-attn
$ uv pip install flash_attn-2.8.1+cu12torch2.7cxx11abiFALSE-cp312-cp312-linux_x86_64.whl

$ uv pip install qwen-vl-utils qwen-omni-utils decord

$ uv pip install msgspec
# 本地安装最新的 ms-swift (通常下载在 /data/hex/projects/ms-swift)
# $ cd /data/hex/projects/ms-swift
$ uv pip install -e .


# megatron 相关
$ uv pip install pybind11 wheel setuptools
$ uv pip install transformer_engine[pytorch] --no-build-isolation

# apex (no uv)
# 和 ms-swift 的文档保持了一致
# $ cd /data/hex/projects/apex
# $ git checkout e13873debc4699d39c6861074b9a3b2a02327f92
# see https://swift.readthedocs.io/en/latest/Megatron-SWIFT/Quick-start.html#environment-setup
$ pip install -v --disable-pip-version-check --no-cache-dir --no-build-isolation --config-settings "--build-option=--cpp_ext" --config-settings "--build-option=--cuda_ext" ./

# megatron-core (core_r0.13.0) (/data/hex/projects/Megatron-LM)
$ git clone --branch core_r0.13.0 https://github.com/NVIDIA/Megatron-LM.git
$ cd Megatron-LM
$ uv pip install -e .
```
