# 09 实战：从零自建 VPN（含踩坑经验）

> ⏱ 适合人群：想自己动手搭的人
> 读完本文你将了解：我的自建踩坑经历、中转的原理、一套适合 TK 直播的网络方案

---

## 一、我的踩坑经历

先说说我自己的经历，希望能帮你少走弯路。

### 🕳️ 第一次踩坑：买了马来西亚 VPS

我之前想做一条马来西亚的专线，想着马来西亚离中国不远，延迟应该可以。

于是买了**荧光云**的一台马来西亚 VPS，结果一测延迟——

> **延迟 200ms+，完全不能用。**

### 🤔 问题出在哪？

后来才知道，马来西亚 VPS 走的是**国际线路**，不是 CN2 也不是专线。

国际线路的意思是：
```
你的请求 → 中国出口 → 绕到美国/欧洲 → 再到马来西亚
```

绕了一大圈，延迟当然高。

### 💡 关键认知：中转很重要

要让马来西亚线路好用，需要加一个**中转**：

```
你 → 中转服务器（香港/日本）→ 马来西亚
```

中转服务器帮你在中间"接一下"，走更优的路径到目标服务器。

### ✅ 最终方案

经过几次调整，最后稳定的方案是：

```
LoongProxy 静态住宅 IP（解决 TK 敏感 IP 问题）
          ↓
香港 CN2 主机（用 [阿里云国际站](https://api.huanghaiwan.com/go/2605280003) 买的，做中转）
          ↓
目标服务器（马来西亚等）
```

### 这样做的效果

| 问题 | 解决前 | 解决后 |
|------|-------|-------|
| 马来西亚延迟 | 200ms+ | 80-120ms |
| TK IP 检测 | 显示机房 IP | 显示住宅 IP |
| 直播间稳定性 | 经常断 | 稳定不断 |
| 晚高峰速度 | 卡顿 | 流畅 |

---

## 二、为什么需要中转

中转的原理很简单：

```
直连：  你 →→→→→→→→→→→ 目标（绕远路）
中转：  你 → 中转服务器 → 目标（走近路）
```

中转服务器要放在**网络枢纽位置**，比如：
- **香港** — 到中国大陆延迟最低（10-30ms），国际出口带宽大
- **日本** — 到中国延迟低（40-60ms），连接欧美也不错
- **新加坡** — 东南亚枢纽

### 中转的三种方式

| 方式 | 说明 | 难度 |
|-----|------|:----:|
| 用机场的中转节点 | 机场已经帮你配好了，直接选节点就行 | ⭐ |
| 用转发服务（如 LoongProxy） | 交钱买转发，配一下就行 | ⭐⭐ |
| 自己买 VPS 做中转 | 自己控制，灵活但需要技术 | ⭐⭐⭐ |

---

## 三、适合 TK 直播的网络拓扑

经过反复调试，以下是我目前用的、比较稳定的方案：

```
                    ┌──────────────────┐
                    │  LoongProxy       │
                    │  静态住宅IP       │
                    │  （目标国家）      │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  香港 CN2 VPS     │
                    │  （阿里云国际）    │
                    │  作为中转转发      │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  你的设备         │
                    │  （手机/电脑）     │
                    │  Clash / 代理配置  │
                    └──────────────────┘
```

### 这个方案的好处

1. **香港 CN2 中转** → 延迟低、稳定
2. **LoongProxy 住宅 IP** → TK 不风控
3. **自己控制** → 不会像机场一样跑路
4. **带宽独享** → 不会跟别人抢

---

## 四、详细搭建步骤

### 准备工作

你需要买：
1. **一台香港 VPS**（推荐 [阿里云国际站](https://api.huanghaiwan.com/go/2605280003) 香港地域，或者 [HostDare](https://api.huanghaiwan.com/go/2605280006) CN2 GIA）
2. **LoongProxy 静态住宅 IP**（目标国家的）

### 第 1 步：购买香港 VPS

我这里以阿里云国际站为例：

1. 访问 [阿里云国际站](https://api.huanghaiwan.com/go/2605280003)（`alibabacloud.com`，不是 `.cn`）
2. 注册账号（需要用海外邮箱）
3. 选择 **ECS** → **香港地域**
4. 配置：
   - 实例：`ecs.t5-lc1m1.small`（1核1G）就够
   - 系统：Ubuntu 22.04
   - 带宽：按量付费或 100Mbps 峰值
5. 付款启动

> 如果你不想买 [阿里云](https://api.huanghaiwan.com/go/2605280003)，[HostDare](https://api.huanghaiwan.com/go/2605280006) 的 CN2 GIA 套餐也可以，$5-8/月。

### 第 2 步：安装 WireGuard（服务端）

连接到你的香港 VPS（用 SSH）：

```bash
# 更新系统
apt update && apt upgrade -y

# 安装 WireGuard
apt install wireguard -y

# 开启 IP 转发
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# 生成服务端密钥
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key
```

创建配置文件 `/etc/wireguard/wg0.conf`：

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <替换为 server_private.key 的内容>

# 注意：这里要加 iptables 转发规则
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

启动 WireGuard：

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### 第 3 步：配置客户端

在你的电脑/手机上生成客户端密钥（或者在 VPS 上生成）：

```bash
# 在本地或 VPS 上生成客户端密钥对
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

在 VPS 的 `/etc/wireguard/wg0.conf` 追加客户端配置：

```ini
[Peer]
PublicKey = <替换为 client_public.key>
AllowedIPs = 10.0.0.2/32
```

然后重启 WireGuard：

```bash
systemctl restart wg-quick@wg0
```

在本地创建客户端配置 `client.conf`：

```ini
[Interface]
PrivateKey = <替换为 client_private.key>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <替换为 VPS 的 server_public.key>
Endpoint = <你的香港 VPS IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### 第 4 步：配置 LoongProxy 转发

在 LoongProxy 后台：
1. 购买目标国家的静态住宅 IP
2. 获取代理信息（IP、端口、账号、密码）
3. 把这些配置到你的 VPS 或客户端

**方案 A：在 VPS 上做转发（推荐）**

在 VPS 上用 redsocks/iptables 把 LoongProxy 的代理转发到 WireGuard 隧道里：

```bash
# 安装 redsocks
apt install redsocks -y
```

配置 `/etc/redsocks.conf`，填入 LoongProxy 的代理信息，然后启动：

```bash
systemctl enable redsocks
systemctl start redsocks
```

**方案 B：在客户端直接配置**

在 Clash Verge 或其他客户端里，直接配置 LoongProxy 的代理作为一个节点使用：

```
类型: SOCKS5
服务器: LoongProxy 给的 IP
端口: LoongProxy 给的端口
用户名: 你的账号
密码: 你的密码
```

### 第 5 步：验证配置

```bash
# 检查 WireGuard 是否运行
wg show

# 测试延迟
ping 10.0.0.1

# 测试外部连接
curl --socks5 127.0.0.1:1080 https://ipinfo.io

# 检查 IP 归属
curl https://ipinfo.io
```

如果显示的是 LoongProxy 的住宅 IP，说明配置成功了。

---

## 五、维护和排错

### 日常维护

```bash
# 查看 WireGuard 状态
wg show

# 查看连接日志
journalctl -u wg-quick@wg0 -f

# 重启 WireGuard
systemctl restart wg-quick@wg0
```

### 常见问题

| 问题 | 原因 | 解决方法 |
|-----|------|---------|
| 连不上 VPS | IP 可能被墙 | 换 IP 或加 CDN 中转 |
| 速度慢 | 中转服务器带宽不够 | 升级带宽或换线路 |
| TK 显示机房 IP | 中转配置不对 | 检查 LoongProxy 是否生效 |
| 晚高峰卡顿 | CN2 带宽拥堵 | 换 CN2 GIA 或加备用线路 |
| 连接不稳定 | 防火墙拦截 | 检查 `ufw` 或 `iptables` 规则 |

---

## 📌 本章小结

```
我的踩坑总结：

1. ❌ 不要直接买马来西亚 VPS 直连 → 国际线路延迟高
2. ❌ 不要以为 VPS 自带 CN2 → 大厂只有阿里云国际站有
3. ✅ 香港 CN2 VPS + 住宅 IP 是最稳的组合
4. ✅ TK 直播必须用住宅 IP，机房 IP 会限流
5. ✅ 先小规模测试，稳定了再扩大

不想折腾？直接买机场省心。
想自己控制？按上面的步骤来。
```

---

**下一章：[故障排查：常见问题与解决方法](./10-troubleshooting.md)**

[← 返回首页](./README.md)
