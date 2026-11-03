# clash-config

基于 YAML 模板和私有覆写配置构建 Mihomo 配置，通过 GitHub Actions 自动校验并发布到 GitHub Secret Gist。

## 设计理念

仓库将配置分为三个部分：

| 文件 | 用途 | Git 中的存储方式 |
|---|---|---|
| `config.yaml` | 公共配置模板，包括基础设置、规则和策略组等 | 明文 |
| `patch/providers.yaml` | 私有代理订阅及 `proxy-providers` 配置 | git-crypt 加密 |
| `patch/override.yaml` | 其他私有配置覆写 | git-crypt 加密 |

构建时，`scripts/merge` 将两个私有 YAML 文件合并到模板中，生成最终配置。GitHub Actions 随后使用 Mihomo 检查配置，并将结果发布到指定 Gist。

主要设计原则：

- **模板与私有配置分离**：公共模板不包含个人订阅、节点信息等私有内容。
- **使用 YAML 合并而非文本 diff**：不依赖上下文敏感的补丁文件，降低模板调整导致合并失败的概率。
- **providers 完整替换**：`patch/providers.yaml` 中的 `proxy-providers` 整体替换模板中的同名配置，保留其 YAML 原始文本结构。
- **override 递归合并**：Mapping 递归合并，List 追加，Scalar 以覆写值为准。
- **Git 透明加密**：通过 git-crypt 自动加密提交内容，工作区仍然使用普通 YAML 文件。
- **自动构建和发布**：提交相关文件后，由 GitHub Actions 执行解密、合并、校验和 Gist 更新。

## 仓库结构

```text
.
├── .github/
│   └── workflows/
│       └── update-gist.yml
├── patch/
│   ├── override.yaml
│   └── providers.yaml
├── scripts/
│   ├── git-crypt-key
│   └── merge
├── .gitattributes
├── .gitignore
├── config.yaml
└── README.md
```

## 部署

### 1. 创建 Secret Gist

在 GitHub 创建一个 Secret Gist：

1. 新建一个 Secret Gist。
2. 添加一个 YAML 文件，例如 `clash_profile.yaml`，初始内容可以为空。
3. 记录 Gist ID。
4. 记录 Gist 文件名。

最终订阅地址可以从 Gist 页面复制 Raw URL。

> [!CAUTION]
>
> Secret Gist 不等于 Private Gist。它不会公开列出或参与常规搜索，但任何获得 Gist URL 的人都可以访问其内容。因此应将 Gist ID 作为秘密妥善保管。

### 2. 创建自己的仓库

建议使用 Private Repository。

本项目的上游仓库是公开仓库。GitHub 的公开 Fork 会保持公开，不能直接将公开 Fork 改为 Private。

如果需要私人部署，请创建一个新的 Private Repository，再复制本项目内容。

例如：

```bash
git clone https://github.com/5kind/clash-config.git
cd clash-config

# Rename the original repository as upstream
git remote rename origin upstream

# Add your own empty private repository
git remote add origin git@github.com:<USER_NAME>/clash-config.git

git push -u origin main
```

将 `<USER_NAME>` 替换为自己的 GitHub 用户名。

### 3. 配置 GitHub Actions Secrets

进入自己的仓库：

`Settings → Secrets and variables → Actions → New repository secret`

添加以下三个 Repository Secrets：

| Secret | 内容 | 用途 |
|---|---|---|
| `GIST_ID` | 自己创建的 Gist ID | 密钥派生及定位 Gist |
| `GIST_FILENAME` | Gist 中的目标文件名 | 指定发布文件 |
| `GIST_PAT` | GitHub Personal Access Token | 更新 Gist |

`GIST_PAT` 使用具有 `gist` 权限的 Personal Access Token。只授予必要权限，不要额外授予仓库读写权限。

### 4. 本地安装依赖

需要：

- Git
- git-crypt
- Python 3
- PyYAML

Debian / Ubuntu：

```bash
sudo apt-get update
sudo apt-get install -y git git-crypt python3 python3-yaml
```

其他 Linux 发行版请安装对应的软件包。

Windows 用户可以通过 WSL 使用同样的 Linux 工具链。

### 5. 解锁私有配置

本项目**不使用** `git-crypt init` 随机生成密钥，而是通过 `scripts/git-crypt-key` 从 `GIST_ID` 确定性派生 git-crypt 兼容密钥。

本地设置环境变量：

```bash
export GIST_ID='你的 Gist ID'
```

生成密钥并解锁仓库：

```bash
KEY_FILE="$(mktemp)"
trap 'rm -f "$KEY_FILE"' EXIT

python3 scripts/git-crypt-key "$KEY_FILE"
git-crypt unlock "$KEY_FILE"
```

解锁后，工作区中的 `patch/override.yaml` 和 `patch/providers.yaml` 会以明文形式呈现。

检查状态：

```bash
git-crypt status
```

注意：`GIST_ID` 必须与创建仓库时使用的值一致。更换 Gist ID 会派生出不同密钥，旧密文不能直接使用新密钥解密。

## 配置维护

### 修改基础模板

直接编辑 `config.yaml`。

适合放入模板的内容包括：

- 通用 DNS 配置
- 通用规则集
- 策略组定义
- 通用代理规则
- 与个人订阅无关的基础设置

提交后，GitHub Actions 会自动构建和发布。

### 更新代理订阅

编辑：

```text
patch/providers.yaml
```

该文件存放完整的 `proxy-providers` 配置。

构建时，合并脚本会使用此文件中的 `proxy-providers` 整体替换模板中的同名配置。

### 修改其他私有配置

编辑：

```text
patch/override.yaml
```

该文件用于存放需要覆盖或追加到模板中的其他配置。

合并规则：

- Mapping：递归合并。
- List：追加到原有列表。
- Scalar：使用覆写文件中的值。

例如：

```yaml
dns:
  proxy-server-nameserver-policy:
    "example.com": "https://dns.example.com/dns-query"
```

可以只在 override 中定义需要追加或覆盖的部分，不必复制整个 `dns` 配置。

### 本地构建

完成 git-crypt 解锁后，可以直接执行：

```bash
python3 scripts/merge \
    config.yaml \
    patch/providers.yaml \
    patch/override.yaml \
    output.yaml
```

默认情况下，脚本也支持不带参数运行，使用内置默认路径。

生成的 `output.yaml` 是合并后的完整配置。

## 自动构建与发布

GitHub Actions 在以下文件发生变化时触发：

- `config.yaml`
- `patch/override.yaml`
- `patch/providers.yaml`
- `.github/workflows/` 下的工作流文件

也可以通过 Actions 页面手动触发 `workflow_dispatch`。

工作流执行过程：

1. Checkout 仓库。
2. 安装 git-crypt 和 PyYAML。
3. 从 `GIST_ID` 生成密钥并解锁私有文件。
4. 合并模板与私有 YAML 配置。
5. 获取 Mihomo 最新版本。
6. 缓存 Mihomo 程序及工作数据。
7. 使用 `mihomo -t` 校验最终配置。
8. 添加提交时间和提交信息。
9. 更新指定 Gist 文件。

## 多人协作

多人协作时，所有需要编辑私有配置的成员必须获得：

- 仓库的相应访问权限。
- 相同的 `GIST_ID`。

每位成员在本地设置自己的 `GIST_ID` 环境变量，并执行密钥生成与解锁步骤。

不需要共享或提交密钥文件。相同的 `GIST_ID` 和固定的派生参数会生成相同的 git-crypt 密钥。

只有负责 Gist 发布的仓库管理员需要配置 `GIST_PAT` 和 `GIST_FILENAME`。普通协作者只需要能够读取和提交仓库，不需要持有发布用的 PAT。

## 安全说明

- 不要提交 `GIST_ID`、`GIST_PAT` 或其他秘密。
- 不要提交本地生成的密钥文件。
- `patch/override.yaml` 和 `patch/providers.yaml` 必须由 git-crypt 管理。
- 新增受保护文件时，先在 `.gitattributes` 中添加加密规则，再执行 `git add`。
- `GIST_ID` 同时用于密钥派生和定位 Gist。获得该 ID 的人既可以访问 Secret Gist，也能够从仓库密文派生解密密钥。
- Secret Gist 不是访问控制机制，不适合存放需要强认证保护的高敏感信息。
- 不要随意修改 `scripts/git-crypt-key` 中的 KDF、Salt、Info 或密钥格式参数，否则会导致已有密文无法正常解密。
- 不要将 `output.yaml` 等构建产物提交到仓库。
