## Qwen-VL模型msmodelslim量化

### 环境

### 下载模型权重和数据集

以qwen3-vl-8B模型为例，可以从 https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct 上下载模型权重。 
以textvqa数据集为例，可在 https://textvqa.org/download/ 下载图片数据集。

```shell
wget https://dl.fbaipublicfiles.com/textvqa/images/train_val_images.zip
```

### modelslim环境安装
可以参考 https://gitcode.com/Ascend/msmodelslim/blob/master/docs/zh/install_guide.md 安装msmodelslim模型压缩工具。

- 这里直接使用打好的镜像:
```shell
swr.cn-south-1.myhuaweicloud.com/ascendhub/cann:8.3.rc1-310p-ubuntu22.04-py3.11
```

- 启动镜像
```shell
docker run -d --ipc=host --network=host --privileged=true --device=/dev/davinci0 --device=/dev/davinci1 --device=/dev/davinci2 --device=/dev/davinci3 --device=/dev/davinci4 --device=/dev/davinci5 --device=/dev/davinci6 --device=/dev/davinci7 --device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc -v /usr/local/sbin/:/usr/local/sbin/ -v /var/log/npu/slog/:/var/log/npu/slog -v /var/log/npu/profiling/:/var/log/npu/profiling -v /var/log/npu/dump/:/var/log/npu/dump -v /var/log/npu/:/usr/slog -v /etc/hccn.conf:/etc/hccn.conf -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi -v /usr/local/dcmi:/usr/local/dcmi -v /usr/local/Ascend/driver:/usr/local/Ascend/driver -v /etc/ascend_install.info:/etc/ascend_install.info -v /etc/vnpu.cfg:/etc/vnpu.cfg --shm-size="250g" -v /data/cjy/:/var/test -it d32643912139 /bin/bash
```

- 安装modelslim
```shell
git clone https://gitcode.com/Ascend/msmodelslim.git
cd msmodelslim
# 注：如果需要进行稀疏量化和压缩，则继续以下操作。
# 3.进入python环境下的site_packages包管理路径，其中${python_envs}为Python环境路径。
cd ${python_envs}/site-packages/msmodelslim/pytorch/weight_compression/compress_graph/  
# 以下是我的环境:
cd /usr/local/python3.11.13/lib/python3.11/site-packages/msmodelslim/pytorch/weight_compression/compress_graph
# 4.编译weight_compression组件，其中${install_path}为CANN软件的安装目录。
sudo bash build.sh ${install_path}/ascend-toolkit/latest
# 以下是我的环境:
bash build.sh /usr/local/Ascend/ascend-toolkit/latest/
# 5.上一步编译操作会得到build文件夹，给build文件夹相关权限
chmod -R 550 build
```

- 安装相关依赖
```shell
pip install transformers -i https://mirrors.aliyun.com/pypi/simple/
pip3 install torch-2.8.0+cpu-cp311-cp311-manylinux_2_28_aarch64.whl -i https://mirrors.aliyun.com/pypi/simple/ 
pip3 install torch_npu-2.8.0-cp311-cp311-manylinux_2_28_aarch64.whl -i https://mirrors.aliyun.com/pypi/simple/
pip install torchvision==0.23.0 -i https://mirrors.aliyun.com/pypi/simple/
```

按照以下步骤启动模型量化，注意参数 model_path 与 save_directory ，前者为原始权重路径，
后者为量化后权重保存路径。如果使用300i服务器量化，参数 device_type 选择cpu：

```shell
cd msmodelslim/example/multimodal_vlm/Qwen3-VL
#train_images_64根据id选取了64张图片
[root@huawei-310p-iduo4-03 Qwen3-VL]# python quant_qwen3vl.py --model_path /var/test/Qwen3-VL-8B-Instruct_64/ --calib_images /var/test/train_images/ --save_directory /var/test/Qwen3-VL-8B-Instruct-W8A8/ --w_bit 8 --a_bit 8 --device_type cpu --trust_remote_code True

```

参考：
- https://gitee.com/hz2901782080/vllm-mindspore_my/blob/my_boom/docs/boom1115/quantization.md