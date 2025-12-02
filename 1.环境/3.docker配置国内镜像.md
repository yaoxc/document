#  sudo: systemctl: command not found

出现 `sudo: systemctl: command not found` 说明你这台 Mac **没有 systemd**，也就 **不需要 `systemctl`**；Docker Desktop on macOS 用 **自己的守护进程**，换源方式跟 Linux 不一样，**照搬 Linux 脚本会报错**。下面给你 **macOS / Windows Docker Desktop 专用 30 秒法**，**一条命令都不用敲**。

------------------------------------------------
macOS（Apple Silicon / Intel 均同）
1. 顶部状态栏 🐳 图标 → **Settings…**  
2. 左侧 **Docker Engine** → 在 JSON 里追加（已有括号就补逗号）
```json
"registry-mirrors": [
  "https://registry.docker-cn.com",
  "https://docker.mirrors.ustc.edu.cn",
  "https://hub-mirrors.c.163.com"
]
```
3. 点 **Apply & Restart** → Docker 自动重启完成。  
4. 验证  
```bash
docker info | grep -i registry
```
能看到刚才的 3 个地址即生效。

------------------------------------------------
Windows Docker Desktop
1. 系统托盘 🐳 → **Settings** → **Docker Engine**  
2. 同样把上面 `"registry-mirrors"` 整块粘进去 → **Apply & Restart** 即可。

------------------------------------------------
一句话  
**macOS/Win 没有 systemctl**，**状态栏图标 → Settings → Docker Engine JSON 里贴镜像地址 → Apply & Restart**，30 秒完成，后续 pull 直接起飞。



把下面整段 JSON 直接复制进去 **Apply & Restart** 即可生效（已帮你加好英文逗号，不报错）。

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirrors.c.163.com"
  ]
}
```

**注意**：  
- 如果原来还有别的字段，只要保证最后一行 **没有尾随逗号**、**整体仍是合法 JSON** 即可。  
- 复制 → 粘贴 → **Apply & Restart** → 国内镜像立刻生效！