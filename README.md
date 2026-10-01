# wireguard-easy — WireGuard 一键部署与接入

把「手动部署 WireGuard 服务端 + 逐台配置客户端」压缩为：
**服务器 1 条命令部署，每台设备 1 条命令/1 次扫码接入**。

```
┌──────────────┐   UDP 51820    ┌────────────────────────────────┐
│ Windows/手机  │ ─────────────► │ 云服务器 (任意 Linux, root)      │
│ 10.0.0.2     │  WireGuard 隧道 │ wg0: 10.0.0.1/24               │
│ 10.0.0.3     │                │ NAT(MASQUERADE) → 互联网        │
└──────────────┘                └────────────────────────────────┘
```

## 快速开始

### ① 服务器（一次性，约 1 分钟）

```bash
git clone <本仓库> && cd wireguard-easy/server
chmod +x *.sh
sudo ./install.sh <你的公网IP或域名>
# 例: sudo ./install.sh vpn.example.com
```

脚本自动完成：装 WireGuard → 生成密钥 → 写配置 → 开转发 → 开机自启。
完成后按提示**到云控制台放行 UDP 51820 入站**（唯一需要手动的一步）。

### ② 添加设备（每台设备 1 条命令）

```bash
sudo ./add-client.sh laptop          # 电脑
sudo ./add-client.sh phone-xiaomi    # 手机
```

自动完成：生成该设备密钥对 → 分配隧道 IP → 服务器登记 → 生成 `.conf` 与**二维码**（输出在 `/root/wireguard-clients/`）。

### ③ 设备接入（两种形态）

| 设备 | 接入方式 |
|---|---|
| **Windows** | 下载 `client/windows/` 到本机，把 `.conf` 放同目录 → 右键**以管理员运行** `onekey-connect.bat` → 自动装客户端、导入、连接 |
| **Android** | 手机装官方 WireGuard App（APK 见 `client/android/README.md`）→ 扫 `add-client.sh` 生成的二维码 → 打开开关 |

> 说明：客户端本体始终是 **WireGuard 官方 App/客户端**（出于安全，不重造轮子）。
> 本仓库封装的是「部署、配密钥、发配置」这些繁琐环节，让整个过程从半小时缩短到 1 分钟。

## 目录结构

```
wireguard-easy/
├── README.md                 ← 本文件
├── server/
│   ├── install.sh            ← 一键部署服务端
│   ├── add-client.sh         ← 一键添加设备（conf + 二维码）
│   ├── list-clients.sh       ← 查看设备
│   └── remove-client.sh      ← 移除设备
├── client/
│   ├── windows/
│   │   ├── onekey-connect.bat    ← 双击/右键管理员运行
│   │   └── onekey-connect.ps1    ← 实际逻辑（装客户端→导入→连接）
│   └── android/
│       └── README.md             ← 手机接入说明（扫码/文件导入）
└── docs/
    ├── MIGRATION.md          ← 换服务器迁移指南（哪些改/哪些复用）
    └── TROUBLESHOOTING.md    ← 排障手册
```

## 设计要点

- **幂等**：install.sh 重复执行不会覆盖已有密钥；add-client.sh 重复执行同设备名会重建
- **安全**：私钥权限 600；客户端私钥只存在于设备与服务器之间传递；每个设备独立密钥对
- **持久化**：peer 热添加的同时写入 `/etc/wireguard/wg0.conf`，重启不丢
- **Endpoint 自动带**：install 时传入的公网 IP/域名会保存，add-client 自动填进客户端配置

## 换服务器？

参见 [docs/MIGRATION.md](docs/MIGRATION.md) —— 结论先行：
**保留服务器私钥迁移，所有客户端只需改一行 Endpoint；否则全部客户端重新导入。**

## 安全提示

- 本仓库所有操作均在 root 下执行，请只在可信服务器上使用
- `/root/wireguard-clients/*.key` 与 `.conf` 含私钥，妥善保管
- 定期 `sudo ./install.sh` 无副作用，可用来检查状态
