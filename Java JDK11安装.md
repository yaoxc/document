# mac下 homebrew 安装Java jdk 11

下面给出 **macOS 上 Homebrew 安装 Java JDK 11** 的最简路径（国内镜像，不翻墙），**30 秒完成**。

1. 一键安装  
   ```bash
   brew install --cask openjdk@11
   ```

2. 让系统找到新版本  
   Homebrew 会把 JDK 装在 `/opt/homebrew/opt/openjdk@11`（Apple Silicon）或 `/usr/local/opt/openjdk@11`（Intel）。  
   把下面这条加到你的 shell 启动文件（`~/.zshrc` 或 `~/.bash_profile`）：  
   ```bash
   export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"
   export JAVA_HOME="/opt/homebrew/opt/openjdk@11"
   ```
   立即生效：  
   ```bash
   source ~/.zshrc   # 或 ~/.bash_profile
   ```

3. 验证  
   ```bash
   java -version
   ```
   输出示例：  
   ```
   openjdk version "11.0.19" 2023-04-18
   OpenJDK Runtime Environment (build 11.0.19+7)
   OpenJDK 64-Bit Server VM (build 11.0.19+7, mixed mode)
   ```
   看到 `11.0.x` 即安装成功。

---

### 常见问题速查
| 问题                      | 解决                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `brew: command not found` | 先装 Homebrew（[https://brew.sh](https://brew.sh) 一键脚本） |
| 仍显示 1.8                | 把 `export JAVA_HOME=...` 放在 **文件末尾** 再 `source`；或 `which java` 确认路径已指向 Homebrew 版 |
| 需要 8/17 共存            | 用 `/usr/libexec/java_home -V` 列出全部版本，按需 `export JAVA_HOME=...` 切换 |

---

一句话：  
`brew install --cask openjdk@11` → 写两行 `export` → `java -version` 看到 11 即完成。