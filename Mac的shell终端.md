在Mac的Shell终端中，`zsh`和`bash`是两种常见的Shell，它们各有特点。以下是它们的主要区别和相关信息：

---

### **1. 默认Shell的变化**
- **macOS Catalina（10.15）及更高版本**：  
  **默认Shell是`zsh`**（Z Shell）。  
  苹果在2019年将默认Shell从`bash`切换为`zsh`，原因是`bash`v3.2（macOS预装版本）过于老旧，且新版本采用GPLv3协议，与苹果的合规要求冲突。

- **macOS Mojave（10.14）及更低版本**：  
  默认Shell是`bash`（版本3.2，2006年发布）。

---

### **2. 主要区别**
| **特性**               | **`zsh`**                          | **`bash`**                     |
|------------------------|------------------------------------|--------------------------------|
| **主题和插件**         | 支持`Oh My Zsh`等框架（海量主题）  | 需手动配置，生态较弱           |
| **配置文件**           | `~/.zshrc`                         | `~/.bash_profile`或`~/.bashrc` |

---

### **3. 如何切换Shell？**
#### **查看当前Shell**
```bash
echo $SHELL
```
#### **切换到`bash`**
```bash
chsh -s /bin/bash  # 需要输入密码
```
#### **切换到`zsh`**
```bash
chsh -s /bin/zsh
```
> **注意**：切换后需重启终端或重新登录生效。

---

### **4. 配置文件路径**
- **`zsh`**：  
  `~/.zshrc`（主配置）、`~/.zprofile`（登录时加载）。
- **`bash`**：  
  `~/.bash_profile`（登录Shell）、`~/.bashrc`（交互式Shell）。

---

### **5. 推荐选择**
- **用`zsh`**：  
  如果你喜欢丰富的主题、插件（如Git集成、语法高亮），或希望使用最新Mac的默认配置。
- **用`bash`**：  
  如果你更熟悉POSIX标准语法，或需要兼容旧脚本（如Linux服务器迁移）。

---

### **6. 其他提示**
- **Homebrew安装的Shell**：  
  可通过`brew install bash`或`brew install zsh`安装新版本（路径通常为`/opt/homebrew/bin/bash`或`/opt/homebrew/bin/zsh`）。
- **脚本兼容性**：  
  编写跨平台脚本时，建议指定`#!/bin/bash`（或`#!/bin/sh`），避免使用`zsh`特有语法。

---

根据你的需求选择即可！大多数用户会受益于`zsh`的现代化特性，尤其是配合`Oh My Zsh`（[官网](https://ohmyz.sh/)）使用时。