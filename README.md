针对 **CentOS Stream** 系统，上轮方案的整体架构不变，但需要额外处理这个发行版特有的两个“关卡”：**Caddy 的安装源**，以及 **SELinux 的权限限制**。以下是针对 CentOS Stream 的完整实操步骤。

---

### 🐍 第一步：确认或安装 Python 3

CentOS Stream 8/9/10 通常已预装 Python 3，可直接使用。如果未安装，通过 dnf 补上：

```bash
# CentOS Stream 8/9/10 通用
sudo dnf install -y python3

# 确认版本（Stream 9 通常为 3.9，Stream 10 通常为 3.12）
python3 --version
```


### 📂 第二步：用 Python 启动静态服务器

进入你的静态文件目录，启动服务并绑定到本地回环地址（安全起见，仅允许 Caddy 在本机访问）：
```bash
# 假设你的静态文件在 /var/www/my-site
cd /var/www/my-site

# 临时设置，终端关闭服务也关闭 
python3 -m http.server 8000 --bind 127.0.0.1

# 终端关闭不影响服务运行 
nohup python3 -m http.server 8000 --bind 127.0.0.1 > /tmp/http8000.log 2>&1 &
```
# 更精细化配置使用系统任务服务配置

一、systemd 服务（推荐，生产用）
写一个 unit 文件 /etc/systemd/system/httpproxy.service：

ini
[Unit]
Description=Python HTTP Server on 8000
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/html
ExecStart=/usr/bin/python3 -m http.server 8000 --bind 127.0.0.1
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
启用并启动：

bash
sudo systemctl daemon-reload
sudo systemctl enable --now httpproxy
sudo systemctl status httpproxy
查看日志：

bash
sudo journalctl -u httpproxy -f
优点：开机自启、崩溃自动重启、日志可查、标准管理方式。把 WorkingDirectory 改成你实际要托管文件的目录，User 改成有读权限的用户。
### 📦 第三步：安装 Caddy（CentOS Stream 专用方式）

CentOS Stream 的默认仓库不包含 Caddy，需要通过 **COPR** 仓库安装：
```bash
# 1. 安装 dnf 插件（COPR 依赖它）
sudo dnf install -y dnf-plugins-core

# 2. 启用 Caddy 官方 COPR 仓库
sudo dnf copr enable -y @caddy/caddy

# 3. 安装 Caddy
sudo dnf install -y caddy
```
安装完成后，Caddy 的 systemd 服务文件会一并就位。

### 📝 第四步：编写 Caddyfile

编辑 `/etc/caddy/Caddyfile`，替换域名和上游地址,：
```caddyfile
{

    email you@exam.com
}

exam.com, www.exam.com {
    reverse_proxy 127.0.0.1:8000
}
```
如果你遇到上游协议协商问题，可以再补上。保存后，先做语法检查：
```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

### ⚠️ 第五步：处理 SELinux（CentOS Stream 的关键步骤）

CentOS Stream 默认开启 SELinux 且处于 `Enforcing` 模式，这会**直接阻止 Caddy 发起网络连接**——包括连接你的 Python 后端，以及连接 Let‘s Encrypt 获取证书。

**必须执行的修复命令：**
```bash
sudo setsebool -P httpd_can_network_connect on
```
这个布尔值允许 `httpd_t` 域（Caddy 服务运行在此域下）发起出站 TCP 连接。

> **一个容易被忽略的陷阱**：Caddy 的 RPM 包在安装脚本中，**可能会主动将该布尔值关闭**（用于限制安装阶段的行为）。因此即使你之前设置过，也建议在启动 Caddy **前重新确认并开启**：
> ```bash
> getsebool httpd_can_network_connect   # 查看当前状态
> # 如果显示 off，再次执行：
> sudo setsebool -P httpd_can_network_connect on
> ```


如果后续仍然遇到权限拒绝，可以用 `ausearch -m avc -ts recent` 查看 SELinux 拒绝日志，必要时用 `audit2allow` 生成策略模块。

### 🔥 第六步：配置防火墙

CentOS Stream 默认使用 `firewalld`，需要放行 80 和 443：
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# 验证
sudo firewall-cmd --list-all
```


### 🚀 第七步：启动并验证

```bash
# 启用开机自启并立即启动
sudo systemctl enable --now caddy

# 如果 Caddy 已经在运行（例如安装时自动启动了），用 reload 应用新配置
sudo systemctl reload caddy

# 查看状态和日志
sudo systemctl status caddy
sudo journalctl -u caddy -f
```
访问 `https://yourdomain.com`，如果证书签发成功且反向代理正常，浏览器应显示安全锁。

**排查顺序建议**：如果 HTTPS 访问失败，按以下顺序检查——
1. `systemctl status caddy` 确认服务是否活跃；
2. `journalctl -u caddy` 查看 Caddy 日志中的 ACME 错误；
3. `getsebool httpd_can_network_connect` 确认 SELinux 布尔值仍为 `on`；
4. `firewall-cmd --list-all` 确认 80/443 已放行；
5. 域名 DNS 解析是否已正确指向服务器。

---

CentOS Stream 与通用 Linux 方案最大的不同，集中在 **COPR 安装源**和 **SELinux 出站连接限制**这两点。把这两步处理好，后续的 HTTPS 自动化流程与上轮完全一致。



** 我们这里是直接占用http和https端口，如果你的服务器上存在多个网站服务，请使用caddy配置网站以及端口转发 **
** 当然，如果你是Nginx master，你也可以使用nginx配置网站和反向代理来部署多个站点解决默认http/https端口占用冲突问题 **
