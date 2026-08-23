# Oracle ARM Sniper

GitHub Actions 定时抢 Oracle Cloud 免费 ARM 实例（VM.Standard.A1.Flex，4 OCPU / 24 GB）。
免费层容量长期被抢光，本工作流在云端持续重试，抢到后开 Issue 通知并自动关掉定时任务。

## 一次性配置

### 1. 拿 Oracle API 密钥

Oracle 控制台 → 右上头像 → **My profile** → 左下 **API keys** → **Add API key**
→ 选 *Generate API key pair* → **Download private key**（这个 `.pem` 只能下一次）→ Add。

弹出的 Configuration file preview 里有三个值，抄下来：`user`、`fingerprint`、`tenancy`。

### 2. 拿子网 ID

控制台 → Networking → **Virtual cloud networks**。没有 VCN 就点
**Start VCN Wizard** → *Create VCN with Internet Connectivity* 一路默认。
进 VCN → Subnets → 点**公有子网**（Public Subnet）→ 复制 **OCID**。

> 必须是公有子网，否则实例拿不到公网 IP，SSH 进不去。

### 3. 生成 SSH 密钥（本机）

```bash
ssh-keygen -t ed25519 -f ~/.ssh/oracle_arm -N ""
```

`~/.ssh/oracle_arm.pub` 的内容填进下面的 `USER_SSH_PUB_KEY`；
`~/.ssh/oracle_arm`（无后缀那个）是登录用的私钥，留在本机。

### 4. 填 Secrets

仓库 → Settings → Secrets and variables → **Actions** → New repository secret，
逐个添加（名字必须完全一致）：

| Secret | 取值 |
|---|---|
| `TENANCY_OCID` | 步骤 1 的 `tenancy`，`ocid1.tenancy.oc1..` 开头 |
| `USER_OCID` | 步骤 1 的 `user`，`ocid1.user.oc1..` 开头 |
| `FINGERPRINT` | 步骤 1 的 `fingerprint`，形如 `a1:b2:c3:...` |
| `REGION` | 你的区域，如 `us-phoenix-1`、`ap-tokyo-1` |
| `PRIVATE_KEY` | 下载的 `.pem` **全文**，含 `-----BEGIN...` / `-----END...` 两行 |
| `SUBNET_ID` | 步骤 2 的公有子网 OCID |
| `USER_SSH_PUB_KEY` | 步骤 3 的 `.pub` 全文，`ssh-ed25519 ...` 开头 |

### 5. 开跑

**定时任务当前是关闭状态**（避免密钥没填齐时空跑）。填完 7 个 Secret 后：

先手动验一次：Actions → 左侧 **Oracle ARM Sniper** → **Run workflow**。
Preflight 步骤会校验全部 7 个密钥并打印解析出的可用域、镜像、子网名。
密钥有错会在这一步直接红叉报错，而不是伪装成"没库存"白跑 45 分钟。

Preflight 绿灯后，再开定时：Actions → Oracle ARM Sniper → 右上 `···`
→ **Enable workflow**。之后每 15 分钟接力，24 小时不断，不用管了。

## 抢到之后

会自动做三件事：

1. 开一个 Issue，标题 `✅ 抢到 Oracle ARM 实例 — <IP>`，正文含公网 IP 和 SSH 命令
2. **关闭定时任务**（避免继续抢出第二台，超出免费额度会计费）
3. run 显示绿勾

登录：

```bash
ssh -i ~/.ssh/oracle_arm ubuntu@<IP>
```

连不上先检查安全列表：VCN → 子网 → Security List → 默认只放行 22 端口，
要开别的端口得自己加 Ingress Rules。

## 免费额度红线

Always Free ARM 额度（2026-08 实测，Oracle 已下调）：

| 资源 | 上限 | 本工作流默认 |
|---|---|---|
| A1 CPU | 2 OCPU | 2（用满） |
| A1 内存 | 12 GB | 12（用满） |
| 块存储 | 200 GB | 系统盘 150 GB |

想改就用 Run workflow 的 `ocpus` / `memory_gb` / `boot_gb` 输入框，不必改代码。

配额已满时脚本会打印 `Service limit reached` 并干净退出，不会空转。

## 重新启用

抢到后定时任务是关掉的。要再抢（比如删了实例想换区域）：
Actions → Oracle ARM Sniper → 右上 `···` → **Enable workflow**。

## 关于封号

Oracle 对免费层试用的态度是：抢实例本身不违规（就是正常的 launch API），
但账号若长期空跑、或被判定滥用，官方有权回收资源。脚本已做节流
（每次 launch 间隔 5s、整轮间隔 30s、遇 429 退避 60s），比手点温和。
真要在意就把 cron 调稀一点。
