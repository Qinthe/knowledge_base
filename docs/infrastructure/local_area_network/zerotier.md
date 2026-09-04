# ZeroTier 组网配置指南（IPv6 直连）

> 适用环境：CentOS Stream 9（服务端）+ Windows 11（客户端）
>
> 最后更新：2026-06-07


## 目录

1. 准备工作：创建虚拟网络
2. CentOS Stream 9 客户端安装与配置
3. Windows 11 客户端安装与配置
4. IPv6 直连配置（推荐方案）
5. 常见问题排查


## 1. 准备工作：创建虚拟网络

### 1.1 注册并登录

访问 [ZeroTier Central](https://my.zerotier.com/)，注册账号并登录。

### 1.2 创建网络

1. 点击 **"Create A Network"** 按钮
2. 系统会生成一个 **16 位** 的网络 ID（例如：`3b19b3a71617a565`）
3. **⚠️ 重要**：立即复制保存这个 Network ID，后续所有设备加入都需要用到

### 1.3 开启 IPv6 分配

1. 点击进入你创建的网络
2. 左侧菜单选择 **"IPv6"**
3. 在 **"IPv6 Auto-Assign"** 下拉菜单中选择 **"Auto-Assign from RFC4193"**
4. 页面会自动保存

> 📌 **说明**：RFC4193 会分配 `fd` 开头的内网专用 IPv6 地址，更安全。

### 1.4 授权管理

- 所有新加入的设备默认状态为 **"NOT AUTHORIZED"**
- 需要在 Members 列表中**手动勾选 Auth? 复选框**才能获得 IP 地址
- **⚠️ 踩坑提醒**：如果设备加入后没有获得 IP，90% 是因为忘记在网页后台勾选授权


## 2. CentOS Stream 9 客户端安装与配置

### 2.1 安装 ZeroTier

```bash
# 官方一键安装脚本
curl -s https://install.zerotier.com | sudo bash

# 启动服务并设置开机自启
sudo systemctl start zerotier-one
sudo systemctl enable zerotier-one

# 验证安装
zerotier-cli status
# 期望输出：200 info xxxxxx 1.x.x ONLINE
```



> ⚠️ **踩坑提醒**：
>
> - 如果 `curl` 命令报 404 错误，可能是官方安装脚本地址变更，改用：
>
>   ```bash
>   curl -s https://raw.githubusercontent.com/zerotier/ZeroTierOne/master/install.sh | sudo bash
>   ```
>
>   
>
> - CentOS Stream 9 安装后如有 `dmesg` 警告（32-bit capabilities），可忽略，不影响使用

### 2.2 加入虚拟网络

```bash
# 替换 <你的网络ID> 为实际的 16 位 ID
sudo zerotier-cli join 3b19b3a71617a565

# 查看是否成功加入
zerotier-cli listnetworks
```



### 2.3 在网页后台授权

1. 登录 [ZeroTier Central](https://my.zerotier.com/)
2. 进入你的网络
3. 在 Members 列表中找到 CentOS 设备（通过 MAC 地址识别）
4. **勾选 Auth? 复选框**
5. 等待几秒，刷新页面，确认 `Managed IPs` 列出现了 IPv4 和 IPv6 地址

### 2.4 开启 IPv6 转发（可选，建议开启）

```bash
# 编辑系统配置文件
sudo vim /etc/sysctl.conf

# 添加以下两行（如果已存在，确保值为 1）
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# 使配置生效
sudo sysctl -p
```



> 📌 **说明**：`ip_forward=1` 对纯点对点访问不是必须的，但 `ipv6.forwarding=1` 有助于 IPv6 直连稳定性。


## 3. Windows 11 客户端安装与配置

### 3.1 下载安装

1. 从 [ZeroTier 官网下载页](https://www.zerotier.com/download/) 下载 Windows 客户端（`.msi` 文件）
2. **右键点击安装包 → 选择"以管理员身份运行"** 进行安装
3. 安装完成后，**务必重启电脑**

> ⚠️ **踩坑提醒**：
>
> - 必须以管理员身份运行安装，否则服务可能无法正常注册
> - 安装后必须重启，否则 ZeroTier 服务可能未完全初始化

### 3.2 加入虚拟网络

**方法一：图形界面**

1. 右键点击系统托盘中的 ZeroTier 图标
2. 选择 **"Join New Network..."**
3. 粘贴你的 16 位 Network ID，点击 Join

**方法二：命令行（管理员）**

```bash
zerotier-cli join 3b19b3a71617a565
```



### 3.3 在网页后台授权

同 CentOS 步骤，在 Members 列表中找到 Windows 设备，**勾选 Auth?**。

### 3.4 检查是否获得 IP

```bash
zerotier-cli listnetworks
```



期望输出类似：

```text
200 listnetworks 3b19b3a71617a565 Tencent_4H4G ... OK PRIVATE ethernet_32776 fd3b:19b3:a716:17a5:.../88,10.90.232.120/24
```



**关键**：`ZT assigned ips` 列必须同时有 IPv4 和 IPv6 地址。


## 4. IPv6 直连配置（推荐方案）

### 4.1 为什么优先用 IPv6？

- IPv6 没有 NAT，直接公网可达，打洞成功率 100%
- 延迟低（通常 20-60ms），不经过海外中转
- 无需搭建 Moon 节点，配置简单

### 4.2 Windows 端配置：强制绑定 IPv6

这是让 IPv6 直连生效的**关键配置**。

**路径**：`C:\ProgramData\ZeroTier\One\local.conf`

**内容**：

```json
{
  "settings": {
    "bind": ["::"]
  }
}
```



**操作步骤**：

1. 右键系统托盘 ZeroTier 图标 → **Exit**（完全退出）
2. 以管理员身份打开记事本，创建/编辑上述文件
3. 粘贴 JSON 内容并保存
4. 重新启动 ZeroTier

> ⚠️ **踩坑提醒**：
>
> - `ProgramData` 是隐藏文件夹，可直接在地址栏输入路径回车进入
> - 文件扩展名必须是 `.conf`，不是 `.txt`
> - 如果之前配置 Moon 时已创建该文件，检查是否被修改或删除
> - **此配置是 IPv6 直连的核心，清理 Moon 时千万不要误删**

### 4.3 CentOS 端配置

CentOS 端**不需要**配置 `local.conf`，默认就会尝试 IPv6 直连。

### 4.4 验证 IPv6 直连

在 CentOS 上执行：

```bash
# 查看对等节点列表
zerotier-cli listpeers
```



**期望结果**：

- Windows 节点的 `<path>` 列显示为 **IPv6 地址**（如 `[240a:42c4:...]:9993`）
- `<latency>` 列有具体的延迟数值（如 `56`），而不是 `-1`
- `<role>` 列显示 `LEAF`

```text
200 listpeers 1e9bce145c [240a:42c4:...]:9993;... 56 1.16.2 LEAF
```



### 4.5 测试连通性

在 CentOS 上 ping Windows 的 IPv6 地址：

```bash
ping -6 fd3b:19b3:a716:17a5:6599:931e:9bce:145c
```



> 📌 需要替换为 Windows 实际的 IPv6 地址（通过 `zerotier-cli listnetworks` 查看）


## Moon 节点（已弃用）

> ZeroTier 官方已将 Moon 标记为 **Deprecated（弃用）**，1.16.x 版本存在 Moon 相关 bug。IPv6 直连已能满足绝大多数场景。如需了解 Moon 的搭建与清理方法，请参阅 [ZeroTier Moon 节点搭建与清理](zerotier-moon.md)。


## 5. 常见问题排查

### 5.1 `zerotier-cli` 命令卡住或无响应

**可能原因**：

- ZeroTier 服务未运行
- 端口 9993 被其他程序（如 Windows IP Helper）占用

**CentOS 解决**：

```bash
sudo systemctl restart zerotier-one
```



**Windows 解决**：

```bash
net stop ZeroTierOneService
net start ZeroTierOneService
```



### 5.2 设备加入网络但没有获得 IP

**原因**：99% 是忘记在 ZeroTier Central 网页后台授权。

**解决**：登录 [my.zerotier.com](https://my.zerotier.com/)，在 Members 列表中勾选 Auth?。

### 5.3 IPv6 能 ping 通，IPv4 不通

**说明**：IPv6 直连已成功，IPv4 打洞失败。这是正常现象，**直接使用 IPv6 地址访问即可**。

### 5.4 IPv6 之前能通，清理 Moon 后不通了

**原因**：清理 Moon 时误删了 `local.conf`，导致 Windows 不再强制使用 IPv6。

**解决**：重新创建 `C:\ProgramData\ZeroTier\One\local.conf`：

```json
{
  "settings": {
    "bind": ["::"]
  }
}
```



重启 ZeroTier 服务。

### 5.5 Windows 自己 ping 自己 IPv6 地址超时

**原因**：Windows IPv6 协议栈损坏。

**解决**：

```bash
netsh int ipv6 reset
netsh winsock reset
# 重启电脑
```



如果仍然无效，重装 ZeroTier 虚拟网卡驱动：

1. 设备管理器 → 网络适配器 → ZeroTier One Virtual Port
2. 右键 → 卸载设备（勾选删除驱动）
3. 重启电脑
4. 重新安装 ZeroTier

### 5.6 CentOS 防火墙导致连接问题

**测试**：

```bash
sudo systemctl stop firewalld
# 测试连接
sudo systemctl start firewalld
```



**永久放行 ZeroTier 网卡**：

```bash
sudo firewall-cmd --permanent --change-interface=zttqh2iatt --zone=trusted
sudo firewall-cmd --reload
```



### 5.7 查看 ZeroTier 监听端口

```bash
# CentOS
netstat -anu | grep :9993

# 期望输出：udp 0 0 0.0.0.0:9993 0.0.0.0:*
```



如果只监听内网 IP（如 `10.0.0.2:9993`），需要配置 `local.conf` 绑定 `0.0.0.0`。


## 附录：常用命令速查

| 命令                                     | 作用                          |
| :--------------------------------------- | :---------------------------- |
| `zerotier-cli status`                    | 查看服务状态                  |
| `zerotier-cli listnetworks`              | 查看已加入的网络及分配的 IP   |
| `zerotier-cli listpeers`                 | 查看所有对等节点及连接类型    |
| `zerotier-cli join <Network ID>`         | 加入网络                      |
| `zerotier-cli leave <Network ID>`        | 离开网络                      |
| `zerotier-cli orbit <Moon ID> <Moon ID>` | 连接 Moon 节点                |
| `zerotier-cli deorbit <Moon ID>`         | 断开 Moon 节点                |
| `zerotier-cli info`                      | 查看本机节点 ID               |
| `systemctl restart zerotier-one`         | 重启 ZeroTier 服务（CentOS）  |
| `net stop/start ZeroTierOneService`      | 重启 ZeroTier 服务（Windows） |


## 最终推荐配置

| 项目                   | 推荐方案                              |
| :--------------------- | :------------------------------------ |
| **网络协议**           | 优先使用 IPv6 直连                    |
| **Windows local.conf** | `{"settings":{"bind":["::"]}}`        |
| **CentOS local.conf**  | 不需要配置                            |
| **Moon 节点**          | 不推荐，IPv6 直连已足够               |
| **防火墙**             | 放行 UDP 9993（IPv4）+ ICMPv6（IPv6） |
| **网络配置文件**       | Windows 虚拟网卡设为"专用网络"        |