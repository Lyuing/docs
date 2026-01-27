## 1. 核心目标与前提

-   **目标**：在云服务器（OpenCloudOS 9）上，**非Docker** 环境直接安装 `Shadowsocks-libev`进行连接和分流

## 2. 服务端安装与配置 (OpenCloudOS 9)

### 2.1 安装必要编译环境


``` bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git gcc automake autoconf libtool make curl gettext-devel pcre-devel mbedtls-devel libsodium-devel libev-devel c-ares-devel
```

### 2.2 编译并安装 Shadowsocks-libev
主目录 (`~`)或 `/tmp`目录中下载源码，完成后进入源码目录编译

```bash
# 克隆源码
git clone https://github.com/shadowsocks/shadowsocks-libev.git
cd shadowsocks-libev
# 更新子模块并编译
git submodule update --init --recursive
./autogen.sh
./configure --disable-documentation
make -j$(nproc)
sudo make install
```

### 2.3 创建服务端配置文件

创建配置文件 `/etc/shadowsocks-libev/config.json`：


```bash
sudo mkdir -p /etc/shadowsocks-libev
sudo vi /etc/shadowsocks-libev/config.json
```

写入以下配置，**需自定义 `server_port` 和 `password`**：


```json
{
    "server": "0.0.0.0",
    "server_port": 8838,
    "password": "YourPassword",
    "method": "chacha20-ietf-poly1305",
    "timeout": 300,
    "mode": "tcp_and_udp"
}
```

### 2.4 创建并启动 Systemd 服务

1.  创建服务文件 `/etc/systemd/system/shadowsocks-libev.service`：
 
    ```bash
        sudo tee /etc/systemd/system/shadowsocks-libev.service << 'EOF'
        [Unit]
        Description=Shadowsocks-libev Default Server Service
        Documentation=man:shadowsocks-libev(8)
        After=network.target
        [Service]
        Type=simple
        ExecStart=/usr/local/bin/ss-server -c /etc/shadowsocks-libev/config.json -u
        User=nobody
        Restart=on-failure
        RestartSec=3s
        LimitNOFILE=32768
        [Install]
        WantedBy=multi-user.target
        EOF
    ```
    
2.  启动并设置开机自启：

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start shadowsocks-libev
    sudo systemctl enable shadowsocks-libev
    sudo systemctl status shadowsocks-libev  # 确认状态为 active (running)
    ```
    
## 3. 防火墙配置（双重保障）

### 3.1 服务器本地防火墙 (firewalld)



```bash
# 放行自定义端口（TCP & UDP）
sudo firewall-cmd --permanent --add-port=8838/tcp
sudo firewall-cmd --permanent --add-port=8838/udp
sudo firewall-cmd --reload
# 验证规则
sudo firewall-cmd --list-ports | grep 8838
```

### 3.2 云服务器安全组（关键）

登录 **云服务器控制台**，找到对应服务器的安全组，添加入站规则：

-   **协议端口**：`TCP:8838` 和 `UDP:8838` （与 `server_port` 一致）
-   **来源**：`0.0.0.0/0` （或根据需求限制IP）
-   **策略**：允许


### 3.3 进行连接测试（验证步骤）

本地电脑上，再次进行最关键的一步测试：

```bash
telnet <服务器IP> 8838
```
Telnet 窗口应变为**空白**（连接成功），而不是显示“连接失败”。


## 4. 客户端配置

在客户端的配置文件中 (`config.yaml`)，添加一个类型为 `ss` (Shadowsocks) 的代理节点。


```yaml
proxies:
  - name: "VPS-SS"  # 节点名称
    type: ss  # 协议类型，必须是 ss
    server: 服务器公网IP  # 例如 43.167.244.44
    port: 8838  # 必须与 server_port 完全一致
    password: "YourPassword"  # 必须与配置文件完全一致
    cipher: chacha20-ietf-poly1305  # 必须与 method 完全一致
    udp: true  # 建议启用

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "VPS-SS"

rules:
  # 示例：AI服务走代理
  - DOMAIN-SUFFIX,openai.com,PROXY
  - DOMAIN-SUFFIX,anthropic.com,PROXY
  - DOMAIN-SUFFIX,github.com,PROXY
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```

## 5. 高效管理

```bash
# 服务管理
sudo systemctl start/stop/restart/status shadowsocks-libev

# 查看实时日志
sudo journalctl -u shadowsocks-libev -f

# 备份配置
sudo cp /etc/shadowsocks-libev/config.json ~/ss-config-backup.json
```

## 6. 连接测试与故障排查
### 6.1 分步验证
1.  **端口监听**：`sudo ss -lnpu | grep 8838` 应显示 `ss-server` 进程。
2.  **网络连通性（关键）** ：在**本地电脑**运行 `telnet <你的服务器IP> 8838`，连接成功会显示黑屏/空白。若失败，**100%为防火墙（安全组）问题**。

### 6.2 常见错误总结
1. `"FirewallD is not running"`说明这台 OpenCloudOS 服务器上默认的防火墙服务没有启动。启动 FirewallD 并配置
```bash
# 1. 启动firewalld服务并设为开机自启
sudo systemctl start firewalld
sudo systemctl enable firewalld

# 2. 现在可以添加端口规则了
sudo firewall-cmd --permanent --add-port=8838/tcp
sudo firewall-cmd --permanent --add-port=8838/udp
sudo firewall-cmd --reload

# 3. 验证规则
sudo firewall-cmd --list-ports | grep 8838
```
