下面给出 **macOS 上一次性装好 Homebrew + Git** 的最简路线（Intel / Apple Silicon 均适用），**不踩坑、不翻墙**。

------------------------------------------------
① 安装 Homebrew（官方一键脚本）
```bash
# 复制整行回车即可（已换国内源，速度快）
/bin/bash -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```
- 脚本会让你选镜像 → 选 **「中科大」或「清华」** → 回车 → 等 2-3 分钟提示 **安装成功** 即可。  
- 最后出现 **Installation successful!** 即可继续下一步。

------------------------------------------------
② 用 Homebrew 装 Git（一条命令）
```bash
brew install git
```
- 首次安装会自动下载依赖，约 30-60 s。  
- 完成后执行 `git --version` 能看到 **git version 2.x.x** 即 OK。

------------------------------------------------
③ 把 Homebrew 和 Git 加入终端环境（通常脚本已自动写好）
如果 `git --version` 提示 **command not found**，再手动执行：
```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```
（Intel Mac 路径是 `/usr/local`；脚本会自动判断）

------------------------------------------------
④ 2 分钟极速验证
| 命令            | 回显示例              | 含义                 |
| --------------- | --------------------- | -------------------- |
| `brew -v`       | Homebrew 4.x.x        | Homebrew 已就绪      |
| `git --version` | git version 2.42.0    | Git 已就绪           |
| `which git`     | /opt/homebrew/bin/git | 系统调用的是 Brew 版 |

------------------------------------------------
⑤ 可选：Git 最小配置（全局）
```bash
git config --global user.name "YourName"
git config --global user.email "you@example.com"
```
------------------------------------------------
一句话总结  
**运行 Gitee 镜像脚本 → `brew install git` → 验证版本**  
全程 2 分钟，Homebrew + Git 一步到位，后续用 `brew install 软件名` 即可任意装包。