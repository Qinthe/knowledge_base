# ZeroTier Moon 节点搭建与清理（已弃用）

> ⚠️ **已弃用**：ZeroTier 官方已不再推荐使用 Moon。如果你的网络已通过 IPv6 直连稳定工作，请直接参考 [ZeroTier 组网配置指南](zerotier.md)，无需阅读本文。

## 目录

1. Moon 节点搭建
2. Moon 节点清理

## 1. Moon 节点搭建

> ⚠️ **官方说明**：ZeroTier 官方已将 Moon 标记为 **"Deprecated"（弃用）**，不再推荐用于新部署。1.16.x 版本存在 Moon 相关 bug，可能导致服务崩溃。如果 IPv6 直连已稳定工作，**强烈建议跳过此章节**。

### 1.1 前提条件

- 一台有**公网 IPv4 地址**的 VPS（本文以腾讯云 VPS 为例）
- VPS 上已安装 ZeroTier 并加入同一虚拟网络
- 需要在网页后台**授权** VPS 设备

### 1.2 开放防火墙端口

**腾讯云安全组**：

- 添加入站规则：**UDP 端口 9993**，来源 `0.0.0.0/0`

**VPS 内部防火墙**：

```bash
sudo firewall-cmd --permanent --add-port=9993/udp
sudo firewall-cmd --reload
```



### 1.3 生成 Moon 配置

在 VPS 上执行：

```bash
cd /var/lib/zerotier-one

# 1. 生成 moon.json
sudo zerotier-idtool initmoon identity.public > moon.json

# 2. 编辑 moon.json，添加公网 IP
sudo sed -i 's/"stableEndpoints": \[\]/"stableEndpoints": ["你的VPS公网IP\/9993"]/' moon.json

# 3. 验证修改
cat moon.json | grep stableEndpoints
# 期望输出："stableEndpoints": ["111.231.21.125/9993"]

# 4. 生成签名文件
sudo zerotier-idtool genmoon moon.json
# 输出：wrote 0000004883afcb68.moon

# 5. 创建目录并移动文件
sudo mkdir -p moons.d
sudo mv 000000*.moon moons.d/

# 6. 修复权限
sudo chown -R zerotier-one:zerotier-one moons.d/

# 7. 重启服务
sudo systemctl restart zerotier-one

# 8. 获取 Moon ID（10位，记下来）
sudo zerotier-cli info | awk '{print $3}'
```



> ⚠️ **踩坑提醒**：
>
> - `stableEndpoints` 必须填 **公网 IP**，不是内网 IP（如 `10.0.0.2`）
> - 端口必须写 `/9993`，不能省略
> - `moons.d/` 目录权限必须属于 `zerotier-one` 用户
> - 如果 VPS 上 ZeroTier 监听 `0.0.0.0:9993` 而不是公网 IP，需要配置 `local.conf` 绑定 `0.0.0.0`

### 1.4 验证 VPS 上 Moon 是否激活

```bash
sudo zerotier-cli listpeers | grep MOON
```



**期望输出**：

```text
200 listpeers 4883afcb68 111.231.21.125/9993;... 10 1.16.2 MOON
```



**如果没有任何输出**，说明 Moon 未生效，检查：

- `moons.d/` 目录下是否有 `.moon` 文件
- `moon.json` 中的 IP 是否正确
- 防火墙是否放行 UDP 9993
- ZeroTier 版本是否为 1.16.x（该版本存在 Moon bug，建议降级到 1.10.6）

### 1.5 客户端连接 Moon

**CentOS 客户端**：

```bash
zerotier-cli orbit 4883afcb68 4883afcb68
sudo systemctl restart zerotier-one
```



**Windows 客户端（管理员 cmd）**：

```bash
zerotier-cli orbit 4883afcb68 4883afcb68
net stop ZeroTierOneService
net start ZeroTierOneService
```



> 📌 为什么输入两次 ID：第一次是 `World ID`，第二次是 `Seed`，自建 Moon 中两者相同。

### 1.6 验证 Moon 是否连接成功

```bash
zerotier-cli listpeers | grep MOON
```



**期望输出**：

```text
200 listpeers 4883afcb68 111.231.21.125/9993;... 45 1.16.2 MOON
```




## 2. Moon 节点清理

如果决定不再使用 Moon，需要彻底清理所有相关配置。

### 2.1 VPS 端清理

```bash
# 1. 停止服务
sudo systemctl stop zerotier-one

# 2. 删除 Moon 配置
sudo rm -rf /var/lib/zerotier-one/moons.d/
sudo rm -f /var/lib/zerotier-one/moon.json
sudo rm -f /var/lib/zerotier-one/*.moon

# 3. 启动服务
sudo systemctl start zerotier-one

# 4. 验证 Moon 已消失
sudo zerotier-cli listpeers | grep MOON
# 应该没有任何输出
```



### 2.2 CentOS 客户端清理

```bash
zerotier-cli deorbit 4883afcb68
sudo systemctl restart zerotier-one
zerotier-cli listpeers | grep MOON
# 应该没有任何输出
```



### 2.3 Windows 客户端清理

```bash
zerotier-cli deorbit 4883afcb68
net stop ZeroTierOneService
net start ZeroTierOneService
zerotier-cli listpeers | findstr MOON
# 应该没有任何输出
```



> ⚠️ **严重踩坑提醒**：
>
> - **不要删除** `C:\ProgramData\ZeroTier\One\local.conf`，这是 IPv6 直连的配置
> - 如果误删了 `local.conf`，IPv6 直连会失效，需要重新创建
> - 清理 Moon 后，如果 IPv6 ping 不通，先检查 `local.conf` 是否存在


