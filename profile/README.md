
VPS 买回来，第一件事很多人都是冲着跑服务去的——装 Nginx、跑 Docker、搭代理、建站。然后有一天，你发现端口死活连不上，服务起了但外面访问不到，折腾半天最后一查，好家伙，是防火墙把你拦住了。

这篇文章就是为这个情况写的。**RackNerd 关闭防火墙**这件事看起来简单，但不同系统有不同做法，临时关和永久关也不是一回事，关了之后怎么配置端口放行才安全，也值得好好说说。顺便，我会介绍 RackNerd 这家提供商目前的套餐和优惠情况——毕竟，防火墙配置再顺畅，也得有台好用的 VPS 才有意义。

---

## RackNerd 是什么，为什么值得关注

RackNerd 是一家美国 IaaS 基础设施服务商，2019 年成立，由数据中心行业老兵 Dustin Cisneros 创办。他们的定位很直白：用接近批发的价格，提供企业级的基础设施。这家公司已经连续多年登上 Inc. 5000 美国最快增长私企榜单，2025 年排名 #94。

他们的 KVM VPS 用的是 Intel Xeon 处理器 + RAID-10 纯 SSD 存储，1Gbps 网口，目前在全球 20+ 个数据中心节点有部署，包括洛杉矶（亚洲优化线路）、新加坡、纽约、阿姆斯特丹等。对国内用户来说，洛杉矶节点提供 CN2 / China Telecom / China Unicom 混合线路，延迟相对友好。

操作系统支持 CentOS、Rocky Linux、AlmaLinux、Fedora、Debian、Ubuntu，也支持通过工单上传自定义 ISO。这对防火墙管理来说很关键——不同系统用的防火墙工具不一样，后面会分别讲。

---

## 为什么 RackNerd VPS 上容易碰到防火墙问题

这不是 RackNerd 特有的问题，是 Linux VPS 的通用情况。装了 Nginx 或者其他服务之后外网访问不到，80/443 端口没反应，大概率是防火墙没放行。

RackNerd 官方博客里也提到过这个场景：ICMP 协议被防火墙拦截会导致 ping 不通，特定端口的连接也会被拒绝。排查网络问题的第一步，就是先把防火墙临时停掉，验证是不是防火墙在作怪。

根据你选择的操作系统不同，防火墙工具也不一样：

- **CentOS 7 / Rocky Linux / AlmaLinux**：默认用 `firewalld`
- **CentOS 6 及以下**：用 `iptables`
- **Ubuntu / Debian**：用 `ufw`

---

## CentOS 7 / Rocky / AlmaLinux：firewalld 操作全指南

### 查看防火墙状态

bash
systemctl status firewalld


或者更简洁的：

bash
firewall-cmd --state


返回 `running` 就是开着的，`not running` 就是关的。

### 临时关闭防火墙（重启后会恢复）

bash
systemctl stop firewalld


适合排查问题时用。关掉之后测试你的服务能不能访问，确认是防火墙问题后再决定是永久关还是配置规则放行。

### 永久关闭防火墙（重启也不会自动开启）

bash
systemctl stop firewalld
systemctl disable firewalld


两条命令分开执行。第一条立即停止，第二条禁止开机自启。

### 验证是否已关闭

bash
systemctl is-enabled firewalld


返回 `disabled` 说明永久关闭成功。

---

### 更推荐的做法：精细化放行端口，而不是直接关防火墙

关掉防火墙是最省事的，但也是最危险的。如果你的 VPS 有对外服务，建议保留防火墙，只开放需要的端口：

bash
# 查看当前规则
firewall-cmd --list-all

# 永久放行 80 端口（HTTP）
firewall-cmd --permanent --add-port=80/tcp

# 永久放行 443 端口（HTTPS）
firewall-cmd --permanent --add-port=443/tcp

# 重载配置让规则生效
firewall-cmd --reload

# 验证端口是否已放行
firewall-cmd --query-port=80/tcp


返回 `yes` 就说明放行成功了。

---

## Ubuntu / Debian：ufw 操作全指南

Ubuntu 默认防火墙工具是 `ufw`，操作方式和 firewalld 略有不同。

### 查看防火墙状态

bash
sudo ufw status


返回 `Status: active` 是开启状态，`Status: inactive` 是关闭状态。

### 临时/永久关闭防火墙

bash
sudo ufw disable


这条命令在 Ubuntu 上直接永久关闭，重启后也不会自动开启。

### 只放行特定端口（推荐）

bash
# 放行 SSH（务必先执行这条，否则可能断连）
sudo ufw allow 22/tcp

# 放行 80 和 443
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 启用防火墙
sudo ufw enable

# 查看规则
sudo ufw status verbose


**重要提醒**：在启用 ufw 之前，一定先放行 22 端口（SSH）。如果你已经改了 SSH 端口，就放行你改后的端口号。不然一旦启用防火墙，你的 SSH 连接就断了，只能通过 VNC 控制台进去修复。

---

## CentOS 6 / 旧系统：iptables 操作指南

老版本系统用的是 iptables，命令风格和 firewalld 完全不同。

### 查看防火墙状态

bash
service iptables status


### 临时关闭

bash
service iptables stop


### 永久关闭（重启不自动开启）

bash
chkconfig iptables off


### 放行端口

bash
# 允许回环接口
iptables -A INPUT -i lo -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许所有出站流量
iptables -A OUTPUT -j ACCEPT

# 放行 SSH（22）
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 放行 HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 拒绝其余未授权访问
iptables -A INPUT -j REJECT

# 保存规则
service iptables save


---

## 防火墙关了还是连不上？试试这几个方向

排查思路按以下顺序走：

**1. 先确认服务本身在监听**

bash
netstat -tlnp | grep 80
# 或
ss -tlnp | grep 80


如果没有输出，说明服务根本没起来，不是防火墙的问题。

**2. 检查网络接口状态**

RackNerd 官方排查文档里提到，可以用 `ip addr` 查看所有网络接口状态，确认接口是 UP 还是 DOWN。

bash
ip addr


如果接口是 DOWN，用 `ifup eth0`（把 eth0 替换成你实际的接口名）把它拉起来。

**3. 通过 VNC 控制台操作**

如果 SSH 连不进去，RackNerd VPS 的 SolusVM 控制面板提供 VNC 控制台入口，可以直接在浏览器里操作系统，不依赖网络连接。找到你的 VPS 后台，打开 VNC 控制台，就能在里面排查防火墙配置或者网络问题了。

---

## RackNerd 当前套餐与价格一览（2026 新年优惠）

说完防火墙配置，也顺便介绍一下目前 RackNerd 在售的套餐。他们目前 2026 新年特惠还在有效期内，价格非常能打。

### KVM VPS（Linux）套餐对比

| 套餐 | vCPU | 内存 | 存储 | 月流量 | 年付价格 | 购买链接 |
|------|------|------|------|--------|----------|----------|
| 1 GB 套餐 | 1 核 | 1 GB | 24 GB SSD | 2000 GB | $11.29/年 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=903&aff=16470) |
| 2 GB 套餐 | 1 核 | 2 GB | 40 GB SSD | 3500 GB | $18.29/年 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=904&aff=16470) |
| 3.5 GB 套餐 ⭐最受欢迎 | 2 核 | 3.5 GB | 65 GB SSD | 7000 GB | $32.49/年 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=905&aff=16470) |
| 4 GB 套餐 | 3 核 | 4 GB | 105 GB SSD | 9000 GB | $43.88/年 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=906&aff=16470) |
| 6 GB 套餐 | 4 核 | 6 GB | 140 GB SSD | 12000 GB | $59.99/年 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=907&aff=16470) |

所有 KVM VPS 套餐均包含：1 个独立 IPv4、1 Gbps 网口、KVM / SolusVM 控制面板、全 Root 权限、免费 Clientexec License、DDoS 防护、多机房可选（下单时选择）。

### 独立服务器套餐对比

| 套餐 | 处理器 | 内存 | 存储 | 带宽 | IP | 月付价格 | 购买链接 |
|------|--------|------|------|------|-----|----------|----------|
| Intel Xeon E3-1240 V3 | 4核/8线程，3.8 GHz Turbo | 32 GB | 1TB SSD + 3TB HDD | 1Gbps 不限流量 | /28（13个可用） | $64.95/月 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=908&aff=16470) |
| AMD Ryzen 7600 | 6核/12线程，5.1 GHz Boost | 64 GB | 1 TB NVMe | 1Gbps 不限流量 | 1 个 IP | $109.95/月 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=909&aff=16470) |
| Dual Intel Xeon E5-2683 v4 | 32核/64线程，3.0 GHz Turbo | 256 GB | 2x 2TB SSD | 1Gbps 不限流量 | /27（29个可用） | $209.00/月 |  [立即购买](https://my.racknerd.com/cart.php?a=add&pid=910&aff=16470) |

独立服务器交付时间为下单后 24-72 小时（手工组装测试）。

---

## 当前有效优惠码

在结账时输入以下优惠码可以享受折扣，选一个最划算的用：

- **INTENSEINVESTOR**：KVM VPS 永久 7 折（月付/年付均有效，持续生效）
- **DRWOOKIEE**：KVM VPS 永久 7 折（效果与上面相同，选其一）
- **WIN-30OFF**：Windows VPS 永久 7 折
- **15OFFDEDI**：独立服务器永久 85 折

👉 [点击此处进入 RackNerd 购买页，结账时填入优惠码](https://my.racknerd.com/aff.php?aff=16470)

注意：一个订单只能使用一个优惠码。新年特惠套餐的年付价格可能已经包含折扣，结账时看一下哪种组合更便宜。

---

## 性价比分析与用户口碑

从社区反馈来看，RackNerd 的用户满意度分布比较两极化：

**口碑好的方面**：
- 价格确实便宜，年付 $11.29 的 1GB KVM VPS 是同类产品里的价格下限
- 续费价格和首购相同，不搞先低后高那套
- 支持工单响应速度快，官方公布平均响应时间不到 10 分钟，社区评价基本吻合
- 洛杉矶节点对国内用户延迟尚可
- 支持支付宝、UnionPay、比特币等支付方式，对国内用户友好

**被提到的缺点**：
- 没有实时客服，只有工单
- 共享主机节点在高峰期偶有降速
- 不是所有套餐都有退款保障（部分套餐有 7 天窗口，购买前确认）
- 独立服务器偶有提速延迟

Trustpilot 评分约 4.2/5（2026 年初数据），在预算型 VPS 提供商里算是比较稳的。

---

## 总结

**RackNerd 关闭防火墙**这件事本身不复杂，核心就是分清楚你的操作系统用的是哪个防火墙工具：

- CentOS 7+：`systemctl stop/disable firewalld`
- Ubuntu/Debian：`sudo ufw disable`
- CentOS 6：`service iptables stop` + `chkconfig iptables off`

不过，关防火墙更多是临时排查手段。生产环境更好的做法是保留防火墙，只把需要的端口放行——既不影响服务访问，也保留了基本防护。

如果你正在找一台价格实惠、配置实在的 VPS 来实践这些操作，RackNerd 目前的新年特惠确实值得看一眼。$11.29 一年，有独立 IP、完整 Root 权限、20+ 机房可选，拿来跑测试、部署小服务、搭自用环境都很合适。

👉 [查看 RackNerd 全部套餐，选一台开始折腾](https://my.racknerd.com/aff.php?aff=16470)
