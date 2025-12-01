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




# 执行source ~/.zshrc   # 或 ~/.bash_profile，报no such file or directory:

报错是因为你的主目录下 **还没有** `.zshrc`（或 `.bash_profile`）这个文件——**不是大问题，自己建一个就行**。  
按你当前用的 shell 二选一执行：

---

### ① 用的是 zsh（macOS Catalina 及以后默认）
```bash
touch ~/.zshrc
```
再把前面的 PATH 内容写进去：
```bash
cat >> ~/.zshrc <<'EOF'
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"
export JAVA_HOME="/opt/homebrew/opt/openjdk@11"
EOF
```
最后让它生效：
```bash
source ~/.zshrc
```

---

### ② 用的是 bash（自己改过）
```bash
touch ~/.bash_profile
cat >> ~/.bash_profile <<'EOF'
export PATH="/opt/homebrew/opt/openjdk@11/bin:$PATH"
export JAVA_HOME="/opt/homebrew/opt/openjdk@11"
EOF
source ~/.bash_profile
```

---

### ③ 确认生效
```bash
java -version
```
看到 `11.0.x` 就 OK 了！

---

一句话：**文件不存在就 `touch` 创建一个再写配置**，然后 `source` 即可。