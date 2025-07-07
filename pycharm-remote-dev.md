# Pycharm远程服务器开发Docker环境搭建

### 0. 前提条件
服务器：运行 Ubuntu（推荐 20.04 或 22.04），已安装 NVIDIA GPU 驱动和 NVIDIA-Docker。

本地电脑：安装 PyCharm Professional 版（社区版不支持远程开发功能）。

SSH 访问：本地电脑可以通过 SSH 访问服务器，且已配置好 SSH 密钥对。

Docker 镜像：使用支持 GPU 的镜像（如 nvidia/cuda 或自定义镜像）。

网络：服务器和本地电脑在同一网络或通过 VPN 可访问。


## 1. 服务器搭建 Docker 环境

### 1.1 我们将创建一个运行在host网络模式下的Docker容器，并启用SSH服务以供PyCharm连接。

准备 Dockerfile： 在服务器上创建项目目录（如 ~/remote_dev）并编写以下 Dockerfile：
```
mkdir ~/remote_dev
cd ~/remote_dev
vim Dockerfile
```

Dockerfile 内容如下：
```
# 使用支持 CUDA 的基础镜像
FROM nvidia/cuda:11.8.0-devel-ubuntu20.04

# 安装 SSH 服务和必要工具
RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    python3-pip \
    sudo \
    && mkdir -p /run/sshd

# 配置 SSH
ARG USER=root
RUN mkdir -p /root/.ssh
COPY my_key.pub /root/.ssh/authorized_keys
RUN chown root:root /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# 安装 Python 依赖（根据需要调整）
RUN pip3 install numpy tensorflow

# 暴露 SSH 端口
EXPOSE 22

# 启动 SSH 服务
CMD ["/usr/sbin/sshd", "-D"]

```

生成 SSH 密钥对： 在本地电脑上生成 SSH 密钥对（如果尚未生成）：
```

ssh-keygen -t rsa -b 4096 -f ~/.ssh/my_key
```

这将生成私钥 my_key 和公钥 my_key.pub。将公钥复制到服务器的 ~/remote_dev 目录：

```
scp ~/.ssh/my_key.pub user@server_ip:~/remote_dev/
```

构建 Docker 镜像： 在服务器的 ~/remote_dev 目录下，构建 Docker 镜像：
```
docker build -t my_gpu_dev .
```

运行 Docker 容器： 使用 host 网络模式启动容器，并启用 GPU 支持：
```
docker run -d --gpus all --network host --name gpu_dev_container my_gpu_dev
```
--gpus all：启用所有 GPU。
--network host：使用 host 网络模式，容器直接使用服务器的网络栈。
-d：后台运行容器。
--name gpu_dev_container：容器名称。

验证容器 SSH 访问： 从本地电脑测试 SSH 连接到服务器的 22 端口（由于使用 host 模式，容器直接使用服务器的 SSH 端口）：
```
ssh -i ~/.ssh/my_key root@server_ip
```

如果成功登录到容器（而不是服务器），说明 SSH 配置正确。

## 2. 配置 PyCharm 远程开发环境

2.1 配置 SSH 连接

打开 PyCharm： 启动 PyCharm Professional 版，进入欢迎界面或现有项目。

添加 SSH 配置：

转到 File > Settings > Tools > SSH Configurations（或 PyCharm > Preferences 在 macOS 上）。

点击 + 添加新配置。

输入以下信息：

Host：服务器的 IP 地址（如 192.168.1.100）。

Port：22（默认 SSH 端口）。

Username：root（因为容器使用 root 用户）。

Authentication type：选择 Key pair (OpenSSH orobligatory) or PuTTY。

Private key file：选择本地的 ~/.ssh/my_key 文件。

Passphrase：如果设置了密钥密码，输入密码。

点击 Test Connection 确保连接成功，然后点击 OK。

配置远程解释器：

转到 File > Settings > Project > Python Interpreter。

点击齿轮图标，选择 Add Interpreter > On SSH。

选择之前创建的 SSH 配置。

在 Interpreter path 中输入容器内的 Python 路径（通常为 /usr/bin/python3）。

在 Sync folders 中设置本地和远程项目文件夹的映射：

Local path：本地项目文件夹路径。

Deployment path：容器内的项目路径（如 /root/project）。

点击 OK，等待 PyCharm 完成解释器自检（可能需要几分钟）。

配置部署：

转到 Tools > Deployment > Configuration。

点击 +，选择 SFTP 类型。

使用相同的 SSH 配置。

设置 Root path 为容器内的项目目录（如 /root/project）。

在 Mappings 选项卡中，配置本地和远程文件夹映射。

启用 Automatic Upload（可选），以便代码更改自动同步到容器。

2.2 测试开发环境

创建测试脚本： 在 PyCharm 中创建一个简单的 Python 脚本（如 test_gpu.py）：

import tensorflow as tf
print("TensorFlow version:", tf.__version__)
print("GPU available:", tf.test.is_gpu_available())

保存并通过 Tools > Deployment > Upload to... 上传到容器。

运行脚本：

点击 PyCharm 的 Run 按钮，脚本将在容器中运行。

检查输出，确认 TensorFlow 检测到 GPU。

调试脚本：

在脚本中设置断点（点击代码行左侧的空白区域）。

点击 Debug 按钮，PyCharm 将通过 SSH 在容器中启动调试会话。

你可以在 PyCharm 中查看变量、堆栈跟踪等调试信息。