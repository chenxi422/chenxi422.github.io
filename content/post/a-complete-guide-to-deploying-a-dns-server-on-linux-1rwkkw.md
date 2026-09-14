---
title: Linux 上部署 DNS 服务器完全指南
slug: a-complete-guide-to-deploying-a-dns-server-on-linux-1rwkkw
url: /post/a-complete-guide-to-deploying-a-dns-server-on-linux-1rwkkw.html
date: '2026-09-14 11:19:04+08:00'
lastmod: '2026-09-14 11:28:17+08:00'
toc: true
isCJKLanguage: true
---



# Linux 上部署 DNS 服务器完全指南

本文覆盖 DNS 基础概念与三种主流方案（BIND9、dnsmasq、CoreDNS）的部署实战，附安全加固与常见问题排查。步骤在 Ubuntu 22.04 / Debian 12 验证通过。

## 一、DNS 基础

DNS（Domain Name System，域名系统）是互联网的"电话簿"，负责把人类易记的域名翻译成机器可识别的 IP 地址。

**主要作用**

|作用|说明|
| --------------| -----------------------------------|
|域名解析|将 `www.example.com`​ 解析为 `93.184.216.34`|
|负载均衡|一个域名对应多个 IP，分流访问压力|
|邮件路由|通过 MX 记录确定邮件投递服务器|
|内网服务发现|通过域名访问内网服务，无需记忆 IP|
|反向解析|通过 IP 反查域名，用于安全审计|

**常见记录类型**

|类型|说明|
| -------| -------------------------|
|A|域名 → IPv4 地址|
|AAAA|域名 → IPv6 地址|
|CNAME|域名别名|
|MX|邮件服务器|
|TXT|文本信息（如 SPF 验证）|
|NS|域名服务器|
|PTR|IP → 域名（反向解析）|

## 二、方案选型

|软件|特点|适用场景|
| ---------| ---------------------------| ------------------------------|
|BIND9|功能最全，业界标准|生产环境、权威 DNS、主从架构|
|dnsmasq|轻量简单，DNS + DHCP 一体|内网、小型环境|
|CoreDNS|插件化、云原生|Kubernetes、现代内网|

## 三、部署前准备：处理 53 端口占用

Ubuntu 默认运行 systemd-resolved，监听 `127.0.0.53:53`​。部署 BIND/dnsmasq 时会报 `address already in use`，这是新手部署失败率最高的问题。

先确认端口占用情况：

```bash
sudo ss -lntup | grep :53
```

专用 DNS 服务器建议直接禁用它：

```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

注意：禁用后本机解析外网依赖 DNS 服务自身的上游转发配置（BIND 的 `forwarders`​、dnsmasq 的 `server=`​、CoreDNS 的 `forward`），务必一并配置，否则会形成解析回环。

## 四、方案一：BIND9

### 4.1 安装

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-dnsutils
```

Ubuntu 20.04 及更早版本的包名为 `bind9utils`。

### 4.2 定义区域

编辑 `/etc/bind/named.conf.local`：

```text
// 正向解析区域
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";
};

// 反向解析区域（对应网段 192.168.1.0/24）
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

### 4.3 正向区域文件

```bash
sudo mkdir -p /etc/bind/zones
```

创建 `/etc/bind/zones/db.example.com`：

```text
$TTL    604800
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2024091001  ; 序列号，每次修改必须递增
                        604800      ; 刷新间隔
                        86400       ; 重试间隔
                        2419200     ; 过期时间
                        604800 )    ; 否定缓存 TTL

; 名称服务器
@       IN      NS      ns1.example.com.

; A 记录
ns1     IN      A       192.168.1.10
www     IN      A       192.168.1.100
mail    IN      A       192.168.1.101

; CNAME 记录
ftp     IN      CNAME   www.example.com.

; MX 记录
@       IN      MX  10  mail.example.com.
```

两个高频错误：

1. `$TTL`​ 的 `$`​ 不能省略，写成 `TTL` 会导致检查工具直接报错
2. 完整域名末尾的 `.`​ 不能漏，否则会被拼接为 `www.example.com.example.com`

### 4.4 反向区域文件

创建 `/etc/bind/zones/db.192.168.1`：

```text
$TTL    604800
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2024091001
                        604800
                        86400
                        2419200
                        604800 )

@       IN      NS      ns1.example.com.

10      IN      PTR     ns1.example.com.
100     IN      PTR     www.example.com.
101     IN      PTR     mail.example.com.
```

### 4.5 配置转发与安全选项

编辑 `/etc/bind/named.conf.options`：

```text
options {
    directory "/var/cache/bind";

    // 本机没有的记录交给上游解析
    forwarders {
        223.5.5.5;
        119.29.29.29;
    };

    allow-query     { any; };                        // 允许谁查询本机数据
    allow-recursion { 127.0.0.1; 192.168.1.0/24; };  // 允许谁走递归，仅内网
    recursion yes;

    allow-transfer  { none; };                       // 禁止区域传送，主从见 4.8

    dnssec-validation auto;
};
```

关键点：`allow-query`​ 控制谁能查数据，`allow-recursion`​ 控制谁能走递归。只写 `allow-query { any; }` 而不限制递归，服务器会暴露为开放递归解析器，被利用发起 DNS 放大攻击。

### 4.6 检查与启动

```bash
# 三条都应返回 OK 或无报错
sudo named-checkconf
sudo named-checkzone example.com /etc/bind/zones/db.example.com
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.192.168.1

# 重启并设置开机自启
sudo systemctl restart bind9
sudo systemctl enable bind9
```

Ubuntu 服务名为 `bind9`​，Debian 为 `named`。

### 4.7 验证

```bash
dig @192.168.1.10 www.example.com       # 正向解析
dig @192.168.1.10 -x 192.168.1.100      # 反向解析
dig @192.168.1.10 www.baidu.com +short  # 转发验证
```

ANSWER SECTION 返回正确 IP 即部署成功。

### 4.8 进阶：主从架构（可选）

生产环境建议一主一从，区域传送用 TSIG 密钥加密。

**1. 主服务器生成密钥**

```bash
sudo tsig-keygen -a hmac-sha256 transfer-key
```

把输出的 key 块保存为主、从两台机器的 `/etc/bind/transfer.key`，并收紧权限：

```bash
sudo chown root:bind /etc/bind/transfer.key
sudo chmod 640 /etc/bind/transfer.key
```

**2. 主服务器 named.conf.local**

```text
include "/etc/bind/transfer.key";

zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";
    allow-transfer { key "transfer-key"; };  // 仅允许持密钥者传送
    also-notify { 192.168.1.11; };           // 从服务器 IP
    notify yes;
};
```

**3. 从服务器（192.168.1.11）named.conf.local**

```text
include "/etc/bind/transfer.key";

server 192.168.1.10 {
    keys { transfer-key; };
};

zone "example.com" {
    type slave;
    masters { 192.168.1.10; };
    file "/var/cache/bind/db.example.com";  // 传送自动生成，勿手改
};
```

**4. 验证同步**

```bash
sudo systemctl restart named
ls /var/cache/bind/
dig @192.168.1.11 www.example.com
```

## 五、方案二：dnsmasq

### 5.1 安装

```bash
sudo apt install -y dnsmasq
```

### 5.2 配置

创建 `/etc/dnsmasq.d/lan.conf`：

```ini
# 本地解析：精确匹配单条记录
host-record=ns1.example.com,192.168.1.10
host-record=www.example.com,192.168.1.100
host-record=mail.example.com,192.168.1.101

# 上游转发：不读取 /etc/resolv.conf，避免回环
no-resolv
server=223.5.5.5
server=119.29.29.29

# 可选：DHCP 向客户端下发本 DNS
# dhcp-option=6,192.168.1.10
```

确认主配置引入了该目录后重启：

```bash
grep -E '^conf-dir=/etc/dnsmasq.d' /etc/dnsmasq.conf || \
  echo 'conf-dir=/etc/dnsmasq.d' | sudo tee -a /etc/dnsmasq.conf

sudo systemctl restart dnsmasq
```

注意：`address=/example.com/IP`​ 会匹配该域及全部子域（近似泛解析），只需精确匹配主机名时用 `host-record`。

### 5.3 验证

```bash
dig @127.0.0.1 www.example.com +short   # 应返回 192.168.1.100
dig @127.0.0.1 www.qq.com +short        # 转发验证
```

## 六、方案三：CoreDNS

### 6.1 安装

```bash
wget https://github.com/coredns/coredns/releases/download/v1.12.0/coredns_1.12.0_linux_amd64.tgz
tar xzf coredns_1.12.0_linux_amd64.tgz
sudo mv coredns /usr/local/bin/
coredns -version
```

新版本请到 GitHub Releases 页面获取对应下载地址。

### 6.2 配置 Corefile

创建 `/etc/coredns/Corefile`：

```text
.:53 {
    hosts {
        192.168.1.100 www.example.com
        192.168.1.101 mail.example.com
        192.168.1.10  ns1.example.com
        fallthrough
    }
    forward . 223.5.5.5 119.29.29.29
    cache 30
    log
    errors
}
```

注意：`hosts`​ 插件不会自动补全区域名，必须写完整域名，只写 `www` 会解析失败。

记录较多时，可改用 `file` 插件加载 BIND 格式的区域文件：

```text
example.com:53 {
    file /etc/coredns/db.example.com
    log
    errors
}

.:53 {
    forward . 223.5.5.5 119.29.29.29
    cache 30
}
```

### 6.3 注册 systemd 服务

创建专用用户：

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin coredns
```

创建 `/etc/systemd/system/coredns.service`：

```ini
[Unit]
Description=CoreDNS DNS server
After=network.target

[Service]
User=coredns
Group=coredns
ExecStart=/usr/local/bin/coredns -conf /etc/coredns/Corefile
Restart=on-failure
RestartSec=5
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now coredns
```

### 6.4 验证

```bash
dig @127.0.0.1 www.example.com +short    # 应返回 192.168.1.100
dig @127.0.0.1 -x 192.168.1.100 +short   # hosts 插件自动生成 PTR 记录
dig @127.0.0.1 www.baidu.com +short      # 转发验证
```

## 七、安全加固清单

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

云服务器还需在控制台安全组放行，尽量只对必要网段开放。

1. 限制递归范围：`allow-recursion` 仅放行内网（见 4.5）
2. 禁止未授权区域传送：`allow-transfer { none; }`，主从场景用 TSIG（见 4.8）
3. 开启响应速率限制：`rate-limit { responses-per-second 10; }`
4. 修改区域文件后必须递增 SOA 序列号，再执行 `sudo rndc reload`
5. 实时日志：`sudo journalctl -u bind9 -f`​（Debian 为 `-u named`）

## 八、客户端配置

临时验证：

```bash
echo "nameserver 192.168.1.10" | sudo tee /etc/resolv.conf
```

`/etc/resolv.conf` 会被 systemd-resolved、NetworkManager 等覆盖，生产环境用 netplan 持久化：

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      nameservers:
        addresses: [192.168.1.10, 223.5.5.5]
        search: [example.com]
```

```bash
sudo netplan apply
```

最省事的方式是由 DHCP 统一下发，客户端零配置（dnsmasq 中启用 `dhcp-option=6`）。

## 九、常见问题排查

|现象|原因|解决方法|
| ------------------------| ---------------------------------| ------------------------------|
|启动失败 `address already in use`|53 端口被 systemd-resolved 占用|见第三节|
|`dig` 超时无响应|53 端口未放行|防火墙与云安全组放行 TCP+UDP|
|返回 `SERVFAIL`|区域文件语法错误|`named-checkzone`​ 检查，`journalctl` 看日志|
|修改记录不生效|SOA 序列号未递增或缓存未过期|递增序列号后 `rndc reload`|
|从服务器不同步|序列号未变或传送被拒|对比主从 SOA，检查 `allow-transfer`|
|返回 `NXDOMAIN`|FQDN 末尾漏了点|检查区域文件中完整域名|
|本机正常，其他机器不行|客户端 DNS 未指向或防火墙拦截|见第八节；`tcpdump -i any port 53` 抓包|

## 十、总结

- 选型：生产权威用 BIND9，内网轻量用 dnsmasq，云原生用 CoreDNS
- 部署核心三步：定义区域 → 编写记录 → 检查启动
- 新手三大坑：`$TTL`​ 漏写 `$`、域名末尾漏点、systemd-resolved 占用 53 端口
- 部署完务必用 `dig` 正反向各验证一遍，生产环境加配从服务器与 TSIG
