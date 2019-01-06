# Reveal 使用

## 1. 查看任意App

需要的东西

- 越狱手机
- Reveal App

### 1.1 越狱

Cydia 后续再补充，或者另外添加文档

### 1.2 环境安装

#### 1.2.1 reveal 安装

xclient 下载 reveal4

#### 1.2.2 安装越狱插件

- 安装 `Cydia Substrate`
- 安装 `OpenSSH`

### 1.3 配置

#### 1.3.1 拷贝 reveal framework

需要把 reveal app 中的 framework 通过 ssh 远程拷贝到手机。

手机和电脑连接到同一个WiFi下，查看手机的IP，修改下面脚本后执行。

IP 样式是 `192.168.x.x`, 下面用 IP 来替代。

```sh
scp /Applications/Reveal.app/Contents/SharedSupport/iOS-Libraries/RevealServer.framework/RevealServer root@IP:/Library/RHRevealLoader/libReveal.dylib
```

默认密码是 `alpine`。

#### 1.3.2 授权错误处理

如果授权失败，显示以下错误

>The authenticity of host '<host>' can't be established.
ECDSA key fingerprint is    SHA256:TER0dEslggzS/BROmiE/s70WqcYy6bk52fs+MLTIptM.
Are you sure you want to continue connecting (yes/no)? yes

需禁用 SSH 远程主机的公钥检查

```sh
cd ~/.ssh
rm known_hosts
vim config
```

在 config 文件中添加以下代码

```sh
Host IP
   StrictHostKeyChecking=no
   UserKnownHostsFile=/dev/null
```

完成后，先测试 SSH ok

```
ssh root@IP
```

如果联通了，再执行 1.3.1 的 `scp` 命令

### 1.4 使用

- 使能需要调试的 App
  - 设置 –> 通用 –> Reveal –> Enabled Applications –> 勾选需要调试的app
- 打开手机上需调试的 App
- 打开Reveal，选择对应的 App