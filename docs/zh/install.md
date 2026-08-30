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
- 能够访问面板，并且 TCP **62050** 端口*从面板一侧*可达，IPv4 或 IPv6 均可——节点
  两者都监听。
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
| `--listen ADDR` | 把控制 API 绑定到 `[::]:62050`（IPv6 通配地址，同时也服务 IPv4）以外的地址 |
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

## 自动安装

面板可以通过 SSH 替你完成上述全部工作。在节点的安装抽屉里——或在添加节点对话框的
底部——切换到 **自动 (SSH)**，把服务器的登录方式交给面板，然后点 **检测**。

它会报告这台服务器是什么（发行版、架构、有没有 systemd 或 Docker）以及上面已经装了
什么，然后给出方案：用哪种部署方式、装到哪个版本。点 **安装**，输出会实时回传。

它和上面那条命令有一个关键区别：它是**上传**的。安装脚本、面板的客户端证书，以及
默认情况下的节点二进制文件，全都沿着这条 SSH 连接下去。服务器不需要通往面板的路由，
不需要通往 GitHub 的路由，也不需要信任面板的 TLS 证书。既然什么都不用下载，安装令牌
也就完全不参与了。

以后再运行一次就是原地升级：`config.json` 和节点自己的证书都保持不动，因此面板固定的
指纹依旧匹配，无需重新注册。

### 凭据

默认什么都不保存。密码只用于那一次连接，随后即被丢弃。

勾选 **在此服务器上授权面板密钥**，面板会把自己的公钥追加到登录用户的
`authorized_keys`，并立刻用第二次连接验证它确实可用。此后的更新完全不需要密码。这是
对该用户全部权限的持久授权——可从节点菜单（**撤销面板密钥**）撤销，或自己从
`~/.ssh/authorized_keys` 里删掉那行 `nexora-panel-…`。

登录用户不必是 root。有 sudo 的用户即可，面板会用你给的密码提权。

第一次连接会固定服务器的 SSH 主机密钥，就像面板固定节点的 TLS 证书一样。若此后服务器
出示了不同的密钥，面板会拒绝连接并明确说明，而不是继续下去：要么机器被换了，要么有人
在你和它之间。

### 把节点迁到另一台服务器

请使用节点菜单里的 **迁移到其他服务器**，而不是直接改地址。有三样东西固定在节点原先
运行的机器上：TLS 证书固定值、SSH 主机密钥，以及面板记住的部署方式。若把它们带到新
服务器，第一样会拒绝所有连接，第二样会拒绝所有登录，第三样会让安装程序为一台可能根本
没有 Docker 的主机安排 Docker 升级。迁移会清除这三样，然后你在新服务器上按全新安装
来做——因为它本来就是新的。

### 部署方式与版本

两者都会自动选定，也都可以覆盖。

部署方式默认沿用服务器现在跑的那一种，因此升级绝不会悄悄把节点从容器变成服务或反过来。
在全新服务器上，有 systemd 就用 systemd，否则用 Docker。显式选择另一种也是支持的：
安装程序会先移除旧的部署，两者不会争抢 62050 端口。

版本默认是面板上已准备好的那个节点二进制文件——面板安装程序在装自己时一并下载的那个。
若指定具体版本，服务器会从 GitHub 下载；面板没有为某个架构准备文件时也会自动走这条路
（面板准备 amd64 和 arm64；armv7、386、s390x 和 riscv64 来自发行版）。Docker 安装则
拉取对应的镜像标签。

## 防火墙

62050 端口使用 mTLS，会拒绝面板以外的所有来源，但没有理由把它暴露给所有人：

```bash
ufw allow from PANEL_IP to any port 62050 proto tcp
```

如果面板是通过 IPv6 访问这个节点的，规则必须写上连接实际来自的那个地址——v4 的规
则并不覆盖 v6 的连接：

```bash
ufw allow from 2001:db8::1 to any port 62050 proto tcp
```

面向客户端的端口是另一回事。它们是分配给该节点的入站所使用的端口，因此在面板中选
定——请在分配模板之后再开放它们，而不是之前。

## 通过 IPv6 访问的节点

两边都不需要任何特殊设置。节点默认绑定 `[::]:62050`，在双栈主机上同样接受 IPv4
连接——所以无论面板走 v4、v6 还是两者兼有，同一套安装都能用。在完全关闭了 IPv6 的
主机上，节点无法绑定该地址，会自行回退到 `0.0.0.0:62050`，而那正是这类主机想要的。

**只有** IPv6 地址的服务器同样不需要额外操作。在面板中直接按原样填写它的地址：

```
2001:db8::1
```

方括号也可以接受，会被去掉——`[2001:db8::1]` 和 `2001:db8::1` 是同一个节点。面板
会在语法需要的地方把方括号加回去，在不需要的地方则不加：分享链接形如
`vless://…@[2001:db8::1]:443?…`，wireguard 配置形如
`Endpoint = [2001:db8::1]:51820`；而 clash、sing-box 的配置以及 OpenVPN 的
`remote` 行携带的是不带方括号的地址。**公开地址**（当它与面板连接所用的地址不同
时，写进链接里的那个地址）以及自动安装的 SSH 主机也遵循同样的规则。

有一点 IPv6 并不会改变：如果入站启用了 TLS 而没有设置 SNI，客户端会用地址本身来
校验证书，因此该地址必须在证书上。请像对待 IPv4 一样，把 IPv6 地址放进 SAN 列表
重新签发节点证书。

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
  "listen": "[::]:62050",
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

节点从面板升级：打开节点的安装抽屉，切换到 **自动 (SSH)**，点 **更新**。面板会上传新的
二进制文件（或拉取新镜像），替换正在运行的那个并重启。节点上不需要敲任何命令。

手动命令同样是原地升级——在已经装有节点的服务器上再运行一次，它会识别现有安装，保留
`config.json` 和证书，只替换二进制文件：

```bash
curl -fsSL https://PANEL/install-node.sh | bash -s -- --panel PANEL --token TOKEN
```

或者自己替换二进制文件并重启：

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
nc -vz -6 2001:db8::1 62050           # ……如果面板通过 IPv6 访问
```

最常见的原因是防火墙规则没有包含面板的地址；其次是云服务商的安全组。如果节点是通
过 IPv6 访问的，请确认防火墙规则是 v6 规则——v4 规则并不覆盖它——并确认
`ss -lnt | grep 62050` 显示节点监听在 `[::]` 而不是 `0.0.0.0`；后者说明主机上的
IPv6 已被关闭，节点做了回退。

**节点被重装后面板拒绝它。** 重装会生成新的服务端证书，而面板仍固定着旧的那一份。
在面板中删除该节点再重新添加——指纹会在下一次首连时重新取得。
