### VS Code Python 虚拟环境配置笔记

记录一下个人在 VS Code 中配置新 Python 项目的标准流程，主要是为了隔离依赖。

**环境:**

- Windows 11
- VS Code
- Python 3.11
- ms-python.python 扩展

#### 1. 初始化项目和 venv

在项目根目录下操作。

```bash
# 1. 创建并进入项目文件夹
mkdir new-py-project && cd new-py-project

# 2. 用 VS Code 打开
code .

# 3. 在 VS Code 终端里创建虚拟环境
# .venv 是约定的文件夹名
python -m venv .venv
```

操作完后，VS Code 会自动检测到 `.venv` 并询问是否选用该解释器，点“是”即可。右下角状态栏会显示当前解释器路径指向了 `.venv`。

#### 2. 激活虚拟环境 & PowerShell 策略问题

创建好后需要激活，才能让终端的 `pip` 等命令指向这个环境。

```powershell
# 激活脚本路径
.\.venv\Scripts\Activate.ps1
```

**注意：首次在 PowerShell 中激活会失败。**

- **原因**: PowerShell 默认的 `ExecutionPolicy` (执行策略) 禁止运行未签名的脚本。
- **解决方法 (一次性)**:
  1. **用管理员权限**打开一个独立的 PowerShell。
  2. 执行 `Set-ExecutionPolicy RemoteSigned`，输入 `Y` 确认。
  3. 此策略允许本地脚本运行，相对安全。

修改策略后，回到 VS Code 终端重新运行激活命令。看到终端提示符前面出现 `(.venv)` 就表示成功了。

#### 3. 安装依赖 (以 numpy 为例)

确保环境已激活（有 `(.venv)` 前缀）。

```bash
# 安装
pip install numpy
# 验证
pip list
```

输出的列表里应该能看到 `numpy` 和它的版本号，这证明包被正确安装到了 `.venv` 里，而不是全局。

#### 4. 测试代码

新建 `main.py` 文件。

```python
import numpy as np

# 简单验证 numpy 是否可用
a = np.arange(15).reshape(3, 5)
print("Numpy is working:")
print(a)
```

在已激活的终端中运行：

```bash
python main.py
```

能正常输出矩阵就代表整个流程没问题。

------

**备忘：** 项目依赖多了之后，记得用 `pip freeze > requirements.txt` 生成依赖清单，方便迁移和协作。