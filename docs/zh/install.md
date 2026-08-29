# 安装 Nexora 节点

节点是数据平面：一个无状态的代理程序，负责终结客户端流量，并执行面板下发的一切。
它没有数据库、没有管理界面，也没有值得备份的配置文件——唯一的状态就是一对证书。
这是刻意为之：丢失的节点在面板里重新注册后会原样回来，节点上没有任何东西需要迁移
或恢复。

你几乎不需要从本页安装节点。**面板会替你安装节点**，它给出的那条单行命令才是受
支持的方式。本页先详细讲这条路径，再讲面板够不到的场景下的手动安装。

## 环境要求

- 一台带 systemd 的 Linux 服务器。amd64、arm64、armv5/v6/v7、386、s390x 和
  riscv64 均有发布。
- root 权限。
- 能够访问面板，并且 TCP **62050** 端口*从面板一侧*可达。
- 分配给该节点的入站所使用的各个端口。

节点不需要自己的许可证密钥。面板的许可证决定它能驱动多少个节点。

## 常规安装

在面板中添加节点，复制它显示的命令：

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- --panel PANEL --token TOKEN
```

在新服务器上以 root 运行。不到一分钟，它会：

1. 识别架构，并**从你的面板**下载节点二进制文件，而不是从互联网下载。
2. 用 `TOKEN` 换取面板的 mTLS 客户端证书。该令牌一次性且有有效期；面板在这次请求
   中就将其作废，之后再也无法使用。
3. 生成节点自己的服务端证书和私钥。
4. 写入 `/opt/nexora-node/config.json` 和一个 systemd unit，然后启动服务。

面板连接上来，在第一次连接时记录节点证书的指纹，此后只接受这一份证书——首次使用
即信任（TOFU）。节点这一侧同样只接受面板的客户端证书。互联网上的其他任何东西都无
法以能通过 TLS 握手的方式与 62050 端口通信。

可以追加到命令的选项：

| 参数 | 作用 |
| --- | --- |
| `--listen ADDR` | 把控制 API 绑定到 `0.0.0.0:62050` 以外的地址 |
| `--binary-url URL` | 从面板以外的位置获取二进制文件 |

如果面板没有为该服务器的架构准备好二进制文件，安装会在第 1 步以
`node binary not provisioned` 失败。请在**面板**主机上准备好它再重新运行——不要
仅仅因为这个就退回手动安装：

```bash
# 在面板服务器上
curl -fsSL -o /tmp/node.tar.gz \
  https://github.com/nexora-vpn/node/releases/latest/download/nexora-node-linux-armv7.tar.gz
tar -C /tmp -xzf /tmp/node.tar.gz
install -m 0755 /tmp/nexora-node/nexora-node /var/opt/nexora/bin/nexora-node-linux-armv7
```

## 防火墙

62050 端口使用 mTLS，会拒绝面板以外的所有来源，但没有理由把它暴露给所有人：

```bash
ufw allow from PANEL_IP to any port 62050 proto tcp
```

面向客户端的端口是另一回事。它们是分配给该节点的入站所使用的端口，因此在面板中选
定——请在分配模板之后再开放它们，而不是之前。

## 面板与节点部署在同一台服务器

**不推荐。** 它能正常工作，Nexora 也不禁止这样做——但在动手之前，请先看看你为什么
多半并不想这么做。

两者并不冲突。面板监听 2095，节点监听 62050；它们分别安装到 `/opt/nexora-panel`
和 `/opt/nexora-node`，作为各自独立的 systemd 服务运行，只共用状态目录的上级路
径——`/var/opt/nexora/nexora.db` 和 `/var/opt/nexora/bin/` 属于面板，
`/var/opt/nexora/certs/` 属于节点。

如果要这么做，就在面板服务器上执行面板给出的那条一行安装命令，并把控制 API 绑定
到回环地址——反正面板就在本机：

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- \
  --panel PANEL --token TOKEN --listen 127.0.0.1:62050
```

然后在面板中用地址 `127.0.0.1` 添加这个节点。62050 端口根本不会离开这台机器，因
此上面的 `ufw` 规则可以完全跳过。

如果用 Docker，面板仓库已经提供了这一部署方式的现成编排：
[`docker/panel-and-node`](https://github.com/nexora-vpn/panel/tree/main/docker/panel-and-node)。

### 为什么不推荐

- **它会公开你面板的地址。** 节点的 IP 会写进你分发的每一条订阅链接和每一份客户
  端配置。合并部署意味着每个用户——以及任何收集这些配置的人——都会知道你的面板在
  哪里。如果该地址日后因承载代理流量而被过滤或拉黑，你失去的不只是一个节点，而是
  面板和全部订阅链接。
- **一台机器，一个命运。** 客户端流量是突发且没有上限的：它会吃掉 CPU、内存、文
  件描述符和带宽。负载下的节点会把面板一起拖垮——而面板一旦宕机，*所有*其他节点
  用户的订阅也会跟着失效。
- **它把一次性的东西和不可替代的东西粘在了一起。** 节点本就是用来丢弃并重新注册
  的；面板的数据库才是整个集群里唯一值得备份的东西。在共用服务器上重装节点，就是
  在这份数据库之上动手。
- **卸载是个坑。** 下面「卸载」一节里的 `rm -rf /var/opt/nexora` 会删掉面板的数据
  库。在共用服务器上只删除 `/opt/nexora-node` 和 `/var/opt/nexora/certs`。
- **它照样占用一个节点名额。** 与面板同机的节点，和其他节点一样计入你许可证的节
  点上限。

对于实验环境、演示，或是你能接受面板与代理共用一个地址、共担一个命运的小型单服务
器部署，这是合理的选择。至于任何丢了会让你心疼的东西，请给面板一台自己的服务器。

## 手动安装

对于无法联网的服务器、没有 systemd 的主机，或者你不希望其对外发起请求的面板，同样
的四步手动完成。

**1. 安装二进制文件。**

```bash
ARCH=amd64   # 或 arm64、armv7、armv6、armv5、386、s390x、riscv64
curl -fsSL -o /tmp/node.tar.gz \
  https://github.com/nexora-vpn/node/releases/latest/download/nexora-node-linux-${ARCH}.tar.gz
tar -C /tmp -xzf /tmp/node.tar.gz
mkdir -p /opt/nexora-node /var/opt/nexora/certs
install -m 0755 /tmp/nexora-node/nexora-node /opt/nexora-node/nexora-node
```

**2. 取得面板的客户端证书。** 在面板的节点安装页面复制它，保存为
`/var/opt/nexora/certs/panel_ca.pem`。这是一份公开证书，不是密钥——拿到它的节点
依然对面板做不了任何事。

**3. 生成节点自己的证书。**

```bash
/opt/nexora-node/nexora-node gencerts -dir /var/opt/nexora/certs
```

**4. 配置并启动。**

```bash
cat > /opt/nexora-node/config.json <<'JSON'
{
  "listen": "0.0.0.0:62050",
  "cert_file": "/var/opt/nexora/certs/ssl_cert.pem",
  "key_file": "/var/opt/nexora/certs/ssl_key.pem",
  "client_ca_file": "/var/opt/nexora/certs/panel_ca.pem"
}
JSON

cp /tmp/nexora-node/nexora-node.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now nexora-node
```

然后在面板中用这台服务器的地址和 62050 端口添加节点。面板会在第一次连接时固定证
书，与脚本安装后的行为完全一致。

## Docker

```bash
git clone https://github.com/nexora-vpn/node
cd node
mkdir -p certs
cp /path/to/panel_ca.pem certs/
docker compose up -d
```

compose 文件刻意使用 host 网络。节点在其入站所使用的端口上终结客户端流量，而那些
端口是之后在面板中选定的——若使用 bridge 网络，每加一个入站就得改一次 compose 文件
并重建容器。

## 升级

节点从面板升级：面板下发新的二进制文件并重启服务。节点上不需要运行任何命令。

手动操作则是替换二进制文件并重启：

```bash
systemctl stop nexora-node
install -m 0755 /tmp/nexora-node/nexora-node /opt/nexora-node/nexora-node
systemctl start nexora-node
```

升级不会动证书，因此面板固定的指纹仍然吻合，节点无需重新注册即可回来。

Docker 下：`docker compose pull && docker compose up -d`。

## 卸载

```bash
systemctl disable --now nexora-node
rm -f /etc/systemd/system/nexora-node.service
systemctl daemon-reload
rm -rf /opt/nexora-node /var/opt/nexora
```

如果面板也装在这台服务器上，**不要**删除 `/var/opt/nexora`——面板的数据库就在那
里。改为只删除 `/var/opt/nexora/certs`。

也请在面板中删除该节点，否则它会继续占用许可证名额，并一直显示为离线。

## 各文件位置

| 路径 | 内容 |
| --- | --- |
| `/opt/nexora-node/nexora-node` | 二进制文件 |
| `/opt/nexora-node/config.json` | 监听地址与证书路径——仅此而已 |
| `/var/opt/nexora/certs/ssl_cert.pem` | 节点自己的服务端证书 |
| `/var/opt/nexora/certs/ssl_key.pem` | 其私钥 |
| `/var/opt/nexora/certs/panel_ca.pem` | 面板的客户端证书，也是唯一被接受的证书 |
| `/etc/systemd/system/nexora-node.service` | 服务 unit |

其余的一切——入站、出站、端点、路由、用户——都由面板下发并保存在内存中。磁盘上没有
别的东西需要备份。

## 排查

**安装命令报 `node binary not provisioned`。** 面板没有该架构的二进制文件。按上文
在面板主机上准备好，然后用新令牌重新运行命令。

**安装命令报 `invalid token` 或 `token already used`。** 节点安装令牌是一次性且有
时限的。在面板中重新生成一个，并重新复制整条命令。

**节点在运行，但面板显示离线。** 两者之间有什么在切断连接。按顺序检查：

```bash
systemctl status nexora-node          # 在节点上
journalctl -u nexora-node -n 50       # 在节点上
nc -vz NODE_IP 62050                  # 从面板服务器
```

最常见的原因是防火墙规则没有包含面板的地址；其次是云服务商的安全组。

**节点被重装后面板拒绝它。** 重装会生成新的服务端证书，而面板仍固定着旧的那一份。
在面板中删除该节点再重新添加——指纹会在下一次首连时重新取得。
