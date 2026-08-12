---
title: 内网Linux机器临时访问外网的优雅方案：Squid HTTP代理
slug: >-
  an-elegant-solution-for-temporarily-accessing-the-external-network-from-an-intranet-linux-immnc
url: >-
  /post/an-elegant-solution-for-temporarily-accessing-the-external-network-from-an-intranet-linux-immnc.html
date: '2026-08-07 11:51:03+08:00'
lastmod: '2026-08-12 16:45:08+08:00'
toc: true
isCJKLanguage: true
---



# 内网Linux机器临时访问外网的优雅方案：Squid HTTP代理

# 

### 场景描述

在内网环境中，我们经常会遇到这样的情况：

- **机器A**（如 `172.25.78.9`）：纯内网机器，无法访问外网。
- **机器B**（如 `172.25.78.18`）：可正常访问外网。

此时，**机器A** 需要运行复杂的安装脚本（如 `oneinstack`​），脚本内部会动态调用 `wget`​、`curl`​ 以及 `dnf/yum` 去外网下载各种依赖包。

### 为什么选择 HTTP 代理？

如果不改网络架构，不修改默认网关，只希望**临时获取外网访问能力，用完即撤**，那么在机器B上搭建一个 HTTP 代理是最佳选择。它侵入性极小，配置简单，清理也极其方便。

---

### 第一步：配置代理服务器（机器B）

在能上网的机器（如 `172.25.78.18`​）上，使用 `Squid` 搭建代理服务。

**1. 安装 Squid**

```bash
sudo dnf install -y squid
```

**2. 配置内网访问权限**  
默认 Squid 只允许本机访问。我们需要修改配置文件，放行整个内网网段。

```bash
# 在配置文件中插入允许 172.25.78.0/24 网段访问的规则
sudo sed -i '/^# Example rule allowing access/a acl localnet src 172.25.78.0/24\nhttp_access allow localnet' /etc/squid/squid.conf
```

**3. 启动服务并放行防火墙**

```bash
sudo systemctl start squid
sudo firewall-cmd --add-port=3128/tcp --permanent
sudo firewall-cmd --reload
```

---

### 第二步：客户端使用代理（机器A）

在需要临时联网的内网机器上，我们需要让命令行工具和包管理器都走代理。

**1. 配置 Shell 环境代理（针对 wget/curl 等）**

```bash
export http_proxy=http://172.25.78.18:3128
export https_proxy=http://172.25.78.18:3128
```

**2. 配置 DNF 包管理器代理（针对 yum/dnf 安装软件）**

```bash
echo "proxy=http://172.25.78.18:3128" | sudo tee -a /etc/dnf/dnf.conf
```

**3. 运行业务脚本**  
此时，该机器已具备完整的外网访问能力，可以直接运行复杂的安装脚本：

```bash
tar xzf oneinstack.tar.gz
cd oneinstack
./install.sh --redis
```

---

### 第三步：用完即撤（清理配置）

当业务安装完毕，不再需要外网时，只需几行命令即可彻底恢复原样，不留痕迹。

**在客户端（机器A）上执行清理：**

```bash
# 1. 清除 Shell 环境变量
unset http_proxy https_proxy

# 2. 清除 DNF 代理配置
sudo sed -i '/^proxy=/d' /etc/dnf/dnf.conf
```

 *(可选)*  **在代理端（机器B）上关闭服务：**

```bash
sudo systemctl disable --now squid
sudo firewall-cmd --remove-port=3128/tcp --permanent
sudo firewall-cmd --reload
```

### 总结

通过 `Squid + 环境变量` 的方式，我们避免了修改路由表和网关带来的潜在网络风险。只需几行命令，就能让内网机器临时“透气”完成复杂软件的安装，最后干净利落地恢复原状。非常适合临时性的运维部署任务。
