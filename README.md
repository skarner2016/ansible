# Ansible 最小示例：Docker 与 SSH 初始化

这个 playbook 在 Ubuntu 24.04 上完成三件事：

- 安装 Docker Engine 和 Docker Compose 插件
- 配置 rootless Docker（与 rootful 共存）
- 把 SSH 端口改成 12322

## 文件说明

| 文件 | 作用 |
|---|---|
| `ansible.cfg` | Ansible 基本配置，运行日志写入 `ansible.log` |
| `inventory.yml` | 只写 `~/.ssh/config` 里的 Host 别名 |
| `server-init.yml` | Docker、rootless Docker、SSH 端口初始化 |

## 1. 确认 Ansible 可用

```bash
ansible --version
```

如果提示 `command not found`：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

想永久生效，把这行加到 `~/.zshrc`。

## 2. 准备 SSH 配置

在 `~/.ssh/config` 里为服务器加一个别名，例如：

```sshconfig
Host tc
    HostName 你的IP
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

然后 `inventory.yml` 只写这个别名：

```yaml
all:
  children:
    servers:
      hosts:
        tc:
```

Ansible 默认使用 OpenSSH，所以 `HostName`、`User`、`Port`、`IdentityFile` 都会从 SSH 配置读取。不要在 inventory 里再写 `ansible_host`、`ansible_user`、`ansible_port`，否则会覆盖 SSH 配置。

## 3. 测试连接

```bash
ssh tc
ansible all -m ping
```

第一次 `ssh tc` 用来接受 host key。看到 `pong` 再继续。

## 4. 执行

```bash
ansible-playbook server-init.yml -K
```

服务器上的 `ubuntu` 用户是免密 sudo 时，可以不加 `-K`：

```bash
ansible-playbook server-init.yml
```

## 5. 验证 SSH

**不要关闭当前 SSH 窗口。** 另开一个终端：

```bash
ssh -p 12322 tc
```

能登录后，把 `~/.ssh/config` 里 `Host tc` 的 `Port` 改成 `12322`。之后直接 `ssh tc` 即可。

## 6. 验证 Docker

```bash
# rootful
sudo docker version
sudo docker info
sudo docker compose version

# rootless（普通用户）
docker version
docker info
docker context show     # 应该是 rootless
docker compose version
```

## Docker 说明

- rootless Docker 默认给 `ubuntu` 用户配置；不是这个用户时，改 `server-init.yml` 里的 `docker_rootless_user`。
- rootful 用 `sudo docker ...`；普通用户直接执行 `docker ...` 默认走 rootless。
- playbook 不会把普通用户加入 `docker` 组，因为该组等价于 root 权限。
- Docker apt 源默认清华 TUNA；可以改 `docker_apt_mirror`，例如阿里云：
  `https://mirrors.aliyun.com/docker-ce/linux/ubuntu`
- 拉镜像慢时，给 `docker_registry_mirrors` 填加速器地址，例如
  `["https://xxxx.mirror.aliyuncs.com"]`。
- rootless 默认不能监听 1024 以下端口。需要 80/443 时，另行设置
  `net.ipv4.ip_unprivileged_port_start=0`。
- Ubuntu 24.04 默认限制非特权 user namespace，playbook 会自动给
  `rootlesskit` 加 AppArmor profile。

## 它做了什么

Docker：

1. 添加 Docker apt 源（国内镜像）和 GPG key。
2. 安装 `docker-ce`、`docker-compose-plugin`、`docker-ce-rootless-extras` 等包。
3. 配置 `/etc/docker/daemon.json`，启动 rootful Docker。
4. 配置 rootless 用户的 subuid/subgid、AppArmor、linger 和 user service。
5. 把普通用户的默认 Docker context 切到 `rootless`。

SSH：

1. 判断 Ubuntu 24.04 是用 `ssh.socket` 还是 `ssh.service` 监听。
2. 修改 sshd 配置和 socket 的监听端口，同时绑定 IPv4 和 IPv6。
3. 用 `sshd -t` 检查语法。
4. 重启 `ssh.socket` 或重载 `ssh.service`。
5. 用 `ss` 确认新端口真的在监听。

## 查看日志

`ansible.cfg` 里配置了：

```ini
log_path = ansible.log
```

运行后可以查看：

```bash
tail -f ansible.log
```

## 回滚 SSH 端口

先确认 `~/.ssh/config` 里的 `Port` 是当前能连通的端口，然后：

```bash
ansible-playbook server-init.yml -e ssh_port=22
```

确认 22 能登录后，再把 `~/.ssh/config` 的 `Port` 改回 22。
