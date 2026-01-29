---
layout: default
title: "利用Cloudflare的Argo隧道在内网服务器或容器内建站"
date: 2022-08-13 10:00:00 +0800
categories: [Linux, 网络技术]
tags: [Cloudflare, Argo, Tunnel, 内网穿透]
---

**更新注意:** Argo 隧道(Argo tunnel)这个名字目前已经被Cloudflare 更名为 **Cloudflare Tunnel**。

Cloudflare提供了一个名为Argo的内网穿透功能，其主要功能类似frp内网穿透工具，但Argo不需要一台有公网IP的主机，只需要一个被Cloudflare托管的域名。

通过Argo可以很方便的在一台内网主机上搭建站点，然后通过绑定域名来访问。假如没有自己的域名，那也没有关系，可以使用Cloudflare分配的随机二级域名。

<!--more-->

同时，Argo也支持http之外的tcp服务，如ssh远程登陆。但是相比frp的方法，ssh等服务的客户端访问比较麻烦。只不过在访问被转发的tcp服务时，客户端需要先运行 `cloudflared access` 命令，才能访问tcp服务。涉及到tcp服务的操作，留到以后的文章再介绍。

**这篇文章只介绍将http服务（网站）通过Argo暴露到公网的内容。**

## Argo隧道的使用方法如下

### 1. 下载并安装程序

在需要搭建网站的主机上下载 `cloudflared` 程序：

*   前往GitHub下载最新版：[https://github.com/cloudflare/cloudflared/releases](https://github.com/cloudflare/cloudflared/releases)

下载后给予运行权限（需要注意的是运行它**不需要**root权限）：

```bash
chmod +x cloudflared
```

---

### 2. 开始配置

这里分为两种情况：**有自己域名** 和 **没有自己域名**。

#### 情况一：有自己域名的情况

1.  **登录 Cloudflare 账户**
    ```bash
    ./cloudflared-linux-amd64 tunnel login
    ```
    此时终端会给出一个URL，复制到浏览器登录并授权。

2.  **创建一个隧道并命名**
    这边范例命名为 `site1`：
    ```bash
    ./cloudflared-linux-amd64 tunnel create site1
    ```

3.  **创建网站的CNAME记录**
    这边范例的域名为 `argo1.example.com`，并将它指向刚才创建的隧道：
    ```bash
    # 其中 site1 为上一步的隧道名
    ./cloudflared-linux-amd64 tunnel route dns site1 argo1.example.com
    ```

4.  **在服务器端运行隧道**
    ```bash
    ./cloudflared-linux-amd64 tunnel run --url localhost:80 site1
    ```

5.  **给搭建的站点绑定域名**
    现在需要将域名 `argo1.example.com` 绑定到你的Web服务上。
    *   这一步有很多做法，最简单的方法是在**宝塔面板**中直接操作，操作完毕后在浏览器访问该域名即可访问站点了。
    *   **注意**：然而默认的情况下站点没有https加密，此时可以在宝塔面板里给站点添加SSL证书，但**不要点选强制HTTPS**。
    *   刷新页面后可以看到HTTPS加好了。

6.  **后台运行**
    最后需要把上面“运行隧道”的命令加入服务（systemd）或者在 `screen` 模式里后台运行，否则终端窗口关闭后隧道通讯就中断了。

#### 情况二：没有自己域名的情况

如果没有自己的域名，可以直接运行以下命令：

```bash
./cloudflared-linux-amd64 tunnel --url localhost:80
```

系统会给你分配一个类似 `this-is-an-example.trycloudflare.com` 的随机域名。

假如你是使用宝塔/aapanel或者其他方式建站的，需要把这个分配的地址绑定上去即可访问。

 
 