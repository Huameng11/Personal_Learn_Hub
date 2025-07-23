# CosyVoice 本地部署详细教程

## 注意事项

+ <font style="color:#DF2A3F;">所有相关的软件、文件名称不要使用中文名称，也不要有中文路径，也不要有空格。  
</font><font style="color:#DF2A3F;">包括C盘用户名，不要有中文或空格。</font>

# 模型部署前准备

+ nvidia显卡,建议显存6G以上
+ AI框架CUDA、cuDNN安装 （已安装可跳过此步骤）
+ Git安装（已安装可跳过此步骤）
+ Miniconda安装（已安装可跳过此步骤）

## 一、AI框架CUDA安装 （已安装可跳过此步骤）

1. 检查本机是否安装CUDA，以及CUDA版本
+ win+R 打开运行，输入cmd打开命令行窗口
+ 输入nvcc -V 查看CUDA版本，注意'V'大写，若提示命令不存在，则未安装CUDA 

```plain
nvcc -V
```

+ 输入NVIDIA-smi，查看当前显卡支持的CUDA版本，最好高于12.0. 

```plain
NVIDIA-smi
```

2. 下载安装CUDA
+ 下载地址：[https://developer.nvidia.com/cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive)
+ 选择合适的版本，这里我选择的是12.4.0，之后依次选择系统windows、x86_64、10、exe（local），自己选择自己对应系统就可以。
+ 点击安装，默认下一步即可，需要时可以更改安装位置，注意路径不要有中文或空格。
+ 配置环境变量， 搜索环境变量设置，编辑环境变量，将cuda的安装位置添加到系统变量。若安装程序已自动添加，无需更改。
3. 下载安装cuDNN
+ 下载地址：[https://developer.nvidia.com/rdp/cudnn-archive](https://developer.nvidia.com/rdp/cudnn-archive)
+ 选择合适的版本，需对应之前安装的CUDA版本，如CUDA版本12.x，下载的对应的v8.9.7。（需要登录NVIDIA账号）
+ <font style="color:#DF2A3F;">免登录下载办法：找到需要的版本，右键-->复制链接-->导入下载器下载或浏览器新建页面粘贴链接下载</font>
+ <font style="color:#DF2A3F;">解压压缩包，将文件夹内所有文件复制至之前安装的CUDA根目录，覆盖替换即可。</font>

```plain
D:\MyToolsSoftWare\CUDADevelopment\
```

+ 配置环境变量
    - 新建cuDNN系统环境变量
    - 变量名：CUDNN。变量值为：CUDA根目录、bin目录、include目录、lib\x64目录，中间由英文分号隔开。

```plain
D:\MyToolsSoftWare\CUDADevelopment;D:\MyToolsSoftWare\CUDADevelopment\bin;D:\MyToolsSoftWare\CUDADevelopment\include;D:\MyToolsSoftWare\CUDADevelopment\lib\x64
```

    - 在系统path变量下，同样添加以上目录
4. 检查安装结果
+ win+R 打开运行，输入cmd打开命令行窗口
+ 输入nvcc -V 查看CUDA版本，注意'V'大写，若能正确返回CUDA版本号，证明安装成功。 

```plain
nvcc -V
```

## 二、Git安装（已安装可跳过此步骤）
+ 下载地址：[https://git-scm.com/downloads](https://git-scm.com/downloads)
+ 选择安装位置，默认安装即可。

## 三、Miniconda安装（已安装可跳过此步骤）
+ 下载地址：[https://docs.anaconda.com/miniconda/](https://docs.anaconda.com/miniconda/)
+ 点击页面中“Miniconda3 Windows 64-bit”版本下载
+ 选择安装位置，建议新建conda文件夹，默认安装，勾选所有选项。
+ 检查安装结果，win+R 打开运行，输入cmd打开命令行窗口
+ 输入conda --version，若能正确返回conda版本号，证明安装成功。 

```plain
conda --version
```

# 部署模型
<font style="color:#DF2A3F;">注意：以下部署过程中命令均在命令行窗口中执行，如果命令行窗口执行过程中，一直提示SSLError或HTTPSConnectionError错误，则表示无法下载，需设置代理端口克隆和下载三方库：</font>

设置方式：在命令行窗口运行以下指令

```plain
set http_proxy=http://127.0.0.1:你的代理端口地址 & set https_proxy=http://127.0.0.1:你的代理端口地址
```

代理端口需自行获取。

## 一、下载项目至本地
1. Git克隆项目文件到本地：

```plain
git clone --recursive https://github.com/FunAudioLLM/CosyVoice.git
cd CosyVoice
git submodule update --init --recursive
```

 PS：国内用户如果克隆失败，可以多尝试几次。有魔法的话，建议开魔法克隆。  
如果仍无法解决，可以下载压缩包文件（时间2024/9/2），历史版本。  
百度网盘下载：[https://pan.baidu.com/s/1lXL6JBZXWFuzHgxUHSzlsg?pwd=1wan](https://pan.baidu.com/s/1lXL6JBZXWFuzHgxUHSzlsg?pwd=1wan) 提取码: 1wan  
夸克网盘下载：[https://pan.quark.cn/s/f8da3aca0d92](https://pan.quark.cn/s/f8da3aca0d92)

2. 创建conda环境
+ 在当前文件夹输入cmd，打开命令行窗口
+ 输入以下命令创建并启动虚拟环境

```plain
conda create -n cosyvoice python=3.8
conda activate cosyvoice
```

## 二、下载安装第三方依赖库
1. 安装前需先修改文件夹中requirements.txt内容

```plain
修改前：onnxruntime-gpu==1.16.0; sys_platform == 'linux'
onnxruntime==1.16.0; sys_platform == 'darwin' or sys_platform == 'windows'
```

```plain
修改后：onnxruntime==1.16.0
```

2. 执行安装命令

```plain
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/ --trusted-host=mirrors.aliyun.com
```

 上边为官方推荐镜像，速度较慢，推荐使用下方镜像。

```plain
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

3. 手动安装torch  
安装过程中torch若下载过慢，可以手动下载该文件后，重新激活虚拟环境，手动安装该库。 
    - 手动下载该文件(可用浏览器、IDM或迅雷下载)，文件地址：[https://download.pytorch.org/whl/cu118/torch-2.0.1%2Bcu118-cp38-cp38-win_amd64.whl](https://download.pytorch.org/whl/cu118/torch-2.0.1%2Bcu118-cp38-cp38-win_amd64.whl)
    - 重新激活虚拟环境，运行手动安装指令：指令格式为  
pip install 下载文件的完整路径 -i [https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple)  
例如：

```plain
pip install D:\AI\torch-2.0.1+cu118-cp38-cp38-win_amd64.whl -i https://pypi.tuna.tsinghua.edu.cn/simple
```

4. 重新执行安装三方库直至全部安装完成

```plain
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

5. <font style="color:#DF2A3F;">可能出现的error</font>
+ cython 安装失败  
解决办法：手动安装

```plain
pip install cython -i https://pypi.tuna.tsinghua.edu.cn/simple
```

+ 各种情况导致的“Failed to build pynini”，pynini安装失败  
解决办法：conda手动安装

```plain
conda install -c conda-forge pynini=2.1.5
```

## 三、下载模型
1. 新建Python程序文件粘贴以下内容保存

```python
from modelscope import snapshot_download
snapshot_download('iic/CosyVoice-300M', local_dir='pretrained_models/CosyVoice-300M')
snapshot_download('iic/CosyVoice-300M-SFT', local_dir='pretrained_models/CosyVoice-300M-SFT')
snapshot_download('iic/CosyVoice-300M-Instruct', local_dir='pretrained_models/CosyVoice-300M-Instruct')
snapshot_download('iic/CosyVoice-ttsfrd', local_dir='pretrained_models/CosyVoice-ttsfrd')
```

2. 激活虚拟环境，直接执行Python程序download_models.py

```plain
python download_models.py
```

3. 也可以从以下链接直接下载模型，解压至项目文件夹即可（2024/9/2）  
    - 百度网盘下载：[https://pan.baidu.com/s/1JDbj8JGKDACVXChe51PaoA?pwd=e8st](https://pan.baidu.com/s/1JDbj8JGKDACVXChe51PaoA?pwd=e8st) 提取码: e8st   
    - 夸克网盘下载：[https://pan.quark.cn/s/fe824caf90a5](https://pan.quark.cn/s/fe824caf90a5)  提取码：VM7V

<font style="color:#DF2A3F;">注意：如果出现模型下载失败问题，可以尝试更新modelscope==1.15.0包到1.17.0，更新脚本如下，进入虚拟环境后粘贴运行即可。</font>

```plain
pip install modelscope==1.17.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

## 四、运行模型
1. 内置音色模型启动(命令行)

```plain
conda activate cosyvoice
python webui.py --port 50000 --model_dir pretrained_models/CosyVoice-300M-SFT
start http://127.0.0.1:50000
```

2. 内置音色模型启动(启动文件)
+ 新建bat文件，把以下命令粘贴进文件，运行即可。

```plain
@echo off
call conda activate cosyvoice
start http://127.0.0.1:50000
python webui.py --port 50000 --model_dir pretrained_models/CosyVoice-300M-SFT
pause
```

3. 克隆音色+跨语种克隆模型启动(命令行)

```plain
conda activate cosyvoice
python webui.py --port 50001 --model_dir pretrained_models/CosyVoice-300M
start http://127.0.0.1:50001
```

4. 克隆音色+跨语种克隆模型启动(启动文件)
+ 新建bat文件，把以下命令粘贴进文件，运行即可。

```plain
@echo off
call conda activate cosyvoice
start http://127.0.0.1:50001
python webui.py --port 50001 --model_dir pretrained_models/CosyVoice-300M
pause
```

4. 内置音色+语气微调模型启动(命令行)

```plain
conda activate cosyvoice
python webui.py --port 50002 --model_dir pretrained_models/CosyVoice-300M-Instruct
start http://127.0.0.1:50002
```

5. 内置音色+语气微调模型启动(启动文件)
+ 新建bat文件，把以下命令粘贴进文件，运行即可。

```plain
@echo off
call conda activate cosyvoice
start http://127.0.0.1:50002
python webui.py --port 50002 --model_dir pretrained_models/CosyVoice-300M-Instruct
pause
```

## 五、总结
根据功能需求，点击对应的.bat文件启动程序。

1. 内置音色生成；
2. 克隆音色+跨语种克隆；
3. 内置音色生成+语气微调；  
浏览器页面会同步打开，但是不显示内容。需等待命令行窗口加载完成后，刷新下网页即可显示程序界面。

参考教程：<font style="color:rgb(24, 25, 28);">https://note.youdao.com/s/Z83Sljd1</font>

