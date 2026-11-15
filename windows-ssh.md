# 1.先检查wsl

```
wsl --status
wsl --version
wsl -l -v
```

如果还没有安装 Ubuntu：
```
wsl --install -d Ubuntu
```

如果提示重启，就重启 Windows。
安装完成后，打开 PowerShell：

```
wsl -d Ubuntu
```

第一次启动时会要求你创建 Linux 用户。

例如：
```
Enter new UNIX username: gpuuser
New password:
Retype new password:
```

这里的 gpuuser 是你自己创建的 Linux 用户名。

注意：
- Linux 用户名可以和 Windows 用户名不同。
- Linux 密码可以和 Windows 密码不同。
- 密码输入时终端不会显示字符，这是 Linux 的正常行为。



# 2 .确认 Ubuntu 是 WSL2

```
wsl -l -v
```
应该看到：
```
  NAME      STATE      VERSION
* Ubuntu    Running    2
```

如果 VERSION 是 1：
```
wsl --set-version Ubuntu 2
```

更新 WSL：
```
wsl --update
```

# 3.配置 WSL2 的 systemd
这一项很重要，因为我们希望 Ubuntu 的 SSH 服务能够开机（即 WSL 实例启动时）自动启动。

先进入 Ubuntu：

```
wsl -d Ubuntu
```

检查：
```
ps -p 1 -o comm=
```

如果输出：
```
systemd
```


说明已经启用，不需要操作。

如果不是 systemd，执行：
```
sudo nano /etc/wsl.conf
```

添加：
```
[boot]
systemd=true
```


如果原来就有其他配置，不要删除，合并即可。

保存退出：
- Ctrl + O
- 回车
- Ctrl + X

然后回到 **Windows PowerShell**：
```
wsl --shutdown
```

重新进入：
```
wsl -d Ubuntu
```

验证：
```
systemctl is-system-running
```


输出 running 或 degraded 都表示 systemd 已经启动，后者还需要进一步检查服务状态。

目前 Ubuntu 新版本通常已经默认启用 systemd。

# 4.配置 Ubuntu 内部的 SSH Server
这是整个配置最关键的部分。

## 安装 SSH Server
在 Ubuntu 中执行：

```
sudo apt update

sudo apt install -y openssh-server
```

注意这里安装的是：
**openssh-server**


不是 **openssh-client**。

## 配置 SSH 端口
我们使用 2222 端口。

创建配置目录：
```
sudo mkdir -p /etc/ssh/sshd_config.d
```

创建配置文件：
```
sudo nano /etc/ssh/sshd_config.d/00-wsl.conf
```

写入以下内容：
```
Port 2222

AddressFamily inet
ListenAddress 0.0.0.0

PubkeyAuthentication yes
PasswordAuthentication yes

PermitRootLogin no
```

配置解释：
配置	作用
Port 2222	SSH 使用 2222 端口
AddressFamily inet	使用 IPv4
ListenAddress 0.0.0.0	监听所有 IPv4 网卡
PubkeyAuthentication yes	允许密钥认证
PasswordAuthentication yes	允许密码认证
PermitRootLogin no	禁止 root 直接 SSH 登录

- Ctrl + O
- 回车
- Ctrl + X

这里先开放密码认证，等你确认可以正常登录后，建议再改成 SSH 密钥认证。

##  检查 SSH 配置
执行：
```
sudo sshd -t
```


如果没有任何输出，说明语法检查通过。

再检查实际生效配置：
```
sudo sshd -T | grep -E '^(port|listenaddress|passwordauthentication|pubkeyauthentication|permitrootlogin)'
```

应该能看到类似：
port 2222
listenaddress 0.0.0.0:2222
pubkeyauthentication yes
passwordauthentication yes
permitrootlogin no


如果这里显示的端口不是 2222，先不要继续，说明配置文件没有按预期生效。

##  启动 SSH 服务
执行：
```
sudo systemctl enable ssh

sudo systemctl start ssh
```

也可以合并为：
```
sudo systemctl enable --now ssh
```

查看状态：
```
sudo systemctl status ssh
```

正常应显示：
Active: active (running)

检查端口：
```
sudo ss -tlnp | grep 2222
```

正常会看到 SSH 监听：
LISTEN 0 128 0.0.0.0:2222

至此，Ubuntu 内部的 SSH Server 已经配置完成。
# 5.配置 WSL 长期运行
这里有一个很容易忽略的问题：

WSL2 不等同于传统 Linux 服务器。

即使你设置了：
```
sudo systemctl enable ssh
```


也不代表 WSL 会永远在后台运行。

微软明确说明，systemd 服务本身不会阻止 WSL 实例自动退出。


如果你把 Ubuntu 终端全部关掉，WSL 可能在空闲一段时间后自动关闭，那么 SSH 就无法连接。

关闭 WSL 空闲自动退出
在 Windows PowerShell 中执行：

```
notepad $env:USERPROFILE\.wslconfig
```


如果文件不存在，记事本会提示创建，选择创建即可。

添加：
```
[general]
instanceIdleTimeout=-1
```


这个配置的作用是关闭 WSL 实例的空闲自动退出。

注意：如果你原本就有 .wslconfig，不要覆盖原有配置，合并进去即可。

然后执行：
```
wsl --shutdown
```

重新启动：
```
wsl -d Ubuntu
```

现在即使关闭 Ubuntu 终端，WSL 实例也不会因为空闲而自动退出。

## nvidia-smi故障分析

这个时候如果nvidia-smi显示报错

```
sjj@HITSZ-SJJ:~$ nvidia-smi Command 'nvidia-smi' not found, but can be installed with: sudo apt install nvidia-utils-580         # version 580.159.03-0ubuntu0.26.04.1, or sudo apt install nvidia-utils-580-server  # version 580.159.03-0ubuntu0.26.04.1 sudo apt install nvidia-utils-595         # version 595.71.05-0ubuntu0.26.04.1 sudo apt install nvidia-utils-595-server  # version 595.71.05-0ubuntu0.26.04.1
```

## **第一步：检查 Windows 是否能识别 RTX 4090**

先退出 Ubuntu：

```
exit
```

打开 Windows PowerShell，执行：

```
nvidia-smi
```

正常应该看到类似：

```
NVIDIA-SMI ...
GPU Name
0  NVIDIA GeForce RTX 4090
```

如果 Windows 能看到 RTX 4090

说明 Windows 显卡驱动基本正常，继续第二步。

如果 Windows 也无法识别，就需要先安装或更新 Windows NVIDIA 驱动。

## **第二步：检查 WSL 是否能找到 NVIDIA 工具**

重新进入 Ubuntu：

```
wsl -d Ubuntu
```

执行：

```
ls -l /usr/lib/wsl/lib/nvidia-smi
```

这里有两种情况。

**情况 A：文件存在**

例如：

```
-r-xr-xr-x ... /usr/lib/wsl/lib/nvidia-smi
```

直接执行：

```
/usr/lib/wsl/lib/nvidia-smi
```

如果能看到 RTX 4090，说明 GPU 已经成功映射到 WSL，只是环境变量 PATH 没有配置好。

执行：

```
echo 'export PATH="$PATH:/usr/lib/wsl/lib"' >> ~/.bashrc

source ~/.bashrc
```

再测试：

```
nvidia-smi
```



## Windows 重启之后怎么办？
instanceIdleTimeout=-1 不代表 Windows 重启后 WSL 会自动启动。

最简单的方式是每次 Windows 开机后执行一次：
```
wsl -d Ubuntu
```


systemd 会自动启动 SSH 服务，之后你可以关闭这个终端。

如果你希望真正做到 Windows 登录后自动启动 WSL，可以使用任务计划程序：
1. 按 Win + R，输入 

   ```
   taskschd.msc
   ```

2. 创建任务，**触发器**设置为当前用户登录时。

3. **操作**选择启动程序。

4. 程序填写：
  ```
  C:\Windows\System32\wsl.exe
  ```


5. 参数填写：
  ```
  -d Ubuntu --exec /bin/sleep infinity
  ```


这样会在用户登录时启动一个后台保持进程，让 WSL 持续运行；SSH 服务由 systemd 自动启动。

这一步是可选的。如果你习惯开机后手动打开一次 Ubuntu，可以暂时不配置。

# 6. 开放服务器到局域网

这里推荐 Windows 11 的 Mirrored Networking（镜像网络模式）。

它允许 WSL 使用镜像网络接口，支持局域网直接访问 WSL 服务。需要 Windows 11 22H2 或更新版本，以及较新的 WSL。

### **1. 修改** `.wslconfig`

再次在 Windows PowerShell 中：

```
notepad $env:USERPROFILE\.wslconfig
```

修改为

```
[general]
instanceIdleTimeout=-1

[wsl2]
networkingMode=mirrored
```

如果之前已经有其他设置，保留并合并。

然后：

```
wsl --shutdown
wsl -d Ubuntu
```

检查 SSH：

```
sudo systemctl status ssh
```

### **2. 配置 Hyper-V 防火墙**

以管理员身份打开 Windows PowerShell：

```
New-NetFirewallHyperVRule `
    -Name "WSL-SSH-2222" `
    -DisplayName "WSL Ubuntu SSH 2222" `
    -Direction Inbound `
    -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' `
    -Protocol TCP `
    -LocalPorts 2222
```

这条规则允许进入 WSL 的 TCP 2222 端口。

### **3. 查看 Windows 局域网 IP**

```
ipconfig
```

假设得到：

```
IPv4 Address: 10.1.1.1
```

在其他电脑上：

```
ssh -p 2222 gpuuser@10.1.1.1
```

就可以进入你的 Ubuntu。

建议在路由器中给 Windows 4090 主机保留固定 IP，避免地址变化。

注意不要在路由器上把这个 SSH 端口转发到公网。只在可信局域网内开放。