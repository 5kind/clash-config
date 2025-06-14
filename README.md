# clash-config
一个简易的clash config模板，github action自动推送更新到gist
 - [config.yaml](./config.yaml) clash config模板，包含规则等，不含订阅；
 - [config.diff.enc](patch/config.diff.enc) 通过`GIST_PAT`加密的patch文件；
 - [encrypt](scripts/encrypt) 运行`encrypt <GIST_PAT>`将`config.diff`加密到`config.diff.enc`，注：如果有[GIST_PAT](GIST_PAT)文件，可以直接运行`encrypt`加密，为用户端调用
 - [decrypt](scripts/decrypt) 运行`decrypt <GIST_PAT>`将`config.diff.enc`解密到`config.diff`，为github action调用
# 使用方法
1. 创建私有 Gist
 - 创建一个 Secret Gist;
 - 在 Gist 中添加一个文件，文件名可以自定，例如 clash_profile.yaml (这个文件名稍后会用作 Secret)。初始内容可以为空;
 - 记下这个 Gist 的 ID (通常是 Gist URL 中你的用户名之后的那串字符);
 - github/<username>/<GIST_ID>/raw/<GIST_FILENAME>为你需要的订阅链接。
2. Fork 并修改此仓库（推荐使用私密仓库）；  
- 仓库中设置 Secrets：
- 进入你的 clash-config 仓库 -> Settings -> Secrets and variables -> Actions -> "New repository secret"
- 添加以下 Secrets：
`GIST_ID`: 你在步骤 1 中获取的 Gist ID。
`GIST_FILENAME`: 你在 Gist 中创建的配置文件的名称 (例如 clash_profile.yaml)。
`GIST_PAT`: 创建一个 GitHub Personal Access Token (PAT)。
访问你的 GitHub Settings -> Developer settings -> Personal access tokens -> Tokens (classic) -> "Generate new token (classic)"。
给 Token 一个描述，选择 Expiration (有效期)。
必须勾选 gist 权限范围，不要勾选其他权限。
生成 Token 后，立即复制它，因为之后无法再次查看。将这个 PAT 粘贴到 `GIST_PAT` 这个 Secret 中。
修改模板与加密patch：
- 修改自用clash config模板到[config.yaml](./config.yaml)；
- 将以上模板与完整config（含订阅）的diff文件到`patch/config.diff`，切记不要将其添加到仓库
- 运行`scripts/encrypt <GIST_PAT>`将`config.diff`加密
- `config.yaml`、`config.diff.enc`需要上传到仓库
push后，github action应当生成正确的gist文件。