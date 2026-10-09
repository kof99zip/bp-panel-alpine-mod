# bp-panel-alpine（容器 / proot 修复版）

Alpine Linux 上的 browser-panel 一键安装启动脚本，针对 **Pterodactyl 容器 + proot** 环境做了修复。

上游/原版：<https://github.com/kuisa/bp-panel-alpine>

---

## 这个版本修的四个问题

### 1. `Cannot find module '../config'`

**症状**

```
Error: Cannot find module '../config'
Require stack:
- .../server/server/db.js
```

**原因**：安装时多套了一层 `server/server/`。`db.js` 里写的是 `require('../config')`，只往上找一层；多一层之后它找的是 `server/config.js`，而真正的 `config.js` 在面板根目录。

**修复**：把 `server/server/*` 提到 `server/`，让结构回到上游布局。

```bash
cd /opt/browser-panel/server
mv server/runtime ./runtime-inner        # runtime 两边同名，先挪开
for f in server/*; do [ "$(basename "$f")" = runtime ] || mv "$f" .; done
cp -a runtime-inner/. runtime/          # 合并 runtime
rm -rf runtime-inner && rmdir server
```

### 2. `EACCES: mkdir '/home/browser/browser-work/node_modules'`

**症状**：路径变成只读的 `/home/browser/...`，尽管 `.env.panel` 里写的是别的路径。

**原因**：`config.js` 读 `process.env.BROWSER_WORK_DIR`，但进程环境里没有 —— `.env.panel` 没被加载，于是退回默认的 `/home/<user>/browser-work`。宿主机上有 systemd 靠 `EnvironmentFile=` 加载，容器里通常没有。

**修复**：新增 `panel-start.sh`，启动前 source 环境变量。

### 3. 工作区被 chown 毒化（容器里最难缠的一个）

**症状**

```
EACCES: permission denied, rmdir '/home/container/browser-work/node_modules/playwright'
```

而且 `chmod 777` 显示成功、权限却纹丝不动，`su browser` 删自己的文件也 `Permission denied`。

**原因**（Pterodactyl + proot 特有）：

- `/home/container` 是**翼龙的主目录挂载**，脚本在它底下按普通 Linux 家目录的规矩写文件，uid 映射对不上
- `browser-launcher.js` 启动时执行 `chown -R browser:browser <workDir>`，把文件改成 `1000:1000`
- 紧随其后的 `chmod -R a+rwX` 本来能救，但 **proot 把 `chmod` 也伪造了**（返回成功、权限不变）
- 于是文件永远停在 `755 1000:1000`，**没有任何进程能再改回来** —— 包括 `su browser` 自己

**修复**：`.env.panel` 里设 `BROWSER_USER=root`，让那句 `chown -R` 变成 `chown -R root:root` —— 对 `0:0` 的文件是空操作，毒化链断掉。

```bash
BROWSER_USER=root
BROWSER_HOME=/root
BROWSER_WORK_DIR=/home/container/bw      # 换新目录，避开已被锁死的旧目录
BROWSER_USER_DATA_DIR=/home/container/bw/persistent
BROWSER_XAUTHORITY=/root/.Xauthority
```

**判断某个路径是否真可写，别看权限位，直接 touch：**

```bash
for d in /opt /root /tmp /home/container /home/container/<sub>; do
  touch "$d/.wt" 2>/dev/null && echo "可写: $d" && rm -f "$d/.wt" || echo "不可写: $d"
done
```

### 4. `setpriv: unrecognized option: reuid=0`

**症状**

```
setpriv: unrecognized option: reuid=0
BusyBox v1.37.0 ...
```

**原因**：Alpine 用 BusyBox 的 `setpriv`，只支持 `-d/--dump`、`--inh-caps` 等，**不认 util-linux 的 `--reuid=` 长选项**。旧版 `browser-launcher.js` 只检查 `command -v setpriv`（BusyBox 存在，检查通过）就直接用长选项，于是炸。

**修复**：本仓库的 `browser-launcher.js` 已包含上游修复 —— 优先 `su-exec`、对 setpriv 做**功能性探测**而非存在性探测、且面板已经是目标用户时**完全跳过降权**。`BROWSER_USER=root` 时 `currentUid===runUid`，日志会出现：

```
[browser-launcher] already running as target user 0:0; skipping su-exec
```

### 5. 面板启动后进程凭空消失

**症状**：`sh panel.sh start` 明明打印「已启动」，几秒后 `pgrep` 查不到，`ss` 里端口也没了，**日志里没有任何报错或崩溃信息**。

**原因**：容器 / tty 环境里 `nohup ... &` 起的子进程，会在父 shell 退出时被连带清理掉。日志安静、进程消失，是这种"会话级清理"的典型特征。

**修复**：`panel.sh` 里 Xvfb 和 node 都改用 `setsid` 完全脱离当前会话，并补 `< /dev/null`：

```sh
setsid "$NODE_BIN" server/index.js >>"$LOG_DIR/panel.log" 2>&1 < /dev/null &
```

---

## 使用方法

### 一键安装

```bash
export BROWSER_USER=root
export BROWSER_HOME=/root
export BROWSER_WORK_DIR=/home/container/bw

bash <(curl -Ls 'https://raw.githubusercontent.com/<用户名>/<新仓库>/main/ins.sh') && \
bash <(curl -Ls 'https://raw.githubusercontent.com/<用户名>/<新仓库>/main/bp.sh') && \
(pkill -f 'node server/index.js' 2>/dev/null || true)

mkdir -p /opt/browser-panel/server/runtime
curl -fLs "https://raw.githubusercontent.com/<用户名>/<新仓库>/main/browser-launcher.js" \
  -o /opt/browser-panel/server/runtime/browser-launcher.js

curl -fLs "https://raw.githubusercontent.com/<用户名>/<新仓库>/main/panel-start.sh" \
  -o /root/panel-start.sh && chmod +x /root/panel-start.sh
```

### 修目录结构（只在那台装错过的机器上跑一次）

```bash
cd /opt/browser-panel/server
mv server/runtime ./runtime-inner
for f in server/*; do [ "$(basename "$f")" = runtime ] || mv "$f" .; done
cp -a runtime-inner/. runtime/
rm -rf runtime-inner && rmdir server
```

### 启动

把 `panel.sh` 放到位（容器里放 `/root/panel.sh`），然后：

```bash
sh panel.sh            # 启动（默认）
sh panel.sh status     # 看进程/端口/工作区是否正常
sh panel.sh restart    # 重启
sh panel.sh stop
sh panel.sh logs       # 跟踪日志
```

它会自动 source `.env.panel`、按 `BROWSER_DISPLAY` 起 Xvfb、再用 `setsid` 拉起面板 —— 容器里不需要 systemd 或 OpenRC。

只想跑 node（已有 systemd 的环境）：

```bash
sh /root/panel-start.sh
```

后台常驻：

```bash
(setsid sh /root/panel.sh start > /dev/null 2>&1 &)
```

---

## 文件说明

| 文件 | 作用 |
|------|------|
| `ins.sh` | 装系统依赖（Chromium、ChromeDriver、Xvfb、Python/Node 包）。本身已支持 `BROWSER_HOME` / `BROWSER_WORK_DIR` |
| `bp.sh` | 配置运行时、用户、OpenRC 服务。**已改**：`prepare_runtime()` 与 OpenRC 段的路径改读环境变量 |
| `panel-start.sh` | **新增**。容器启动器，负责 source `.env.panel` 再拉起 node |
| `panel.sh` | **容器主启动脚本**（已重写）。不依赖 systemd/OpenRC，自动加载 `.env.panel`、拉起 Xvfb 和面板，支持 `start` / `stop` / `restart` / `status` / `logs` |
| `browser-launcher.js` | 任务启动器。含上游 setpriv/su-exec 修复，**另加** `BROWSER_HOME` 环境变量兜底 |
| `ff.sh(ruyipage+ff无法运行)` | 已废弃，Firefox + ruyipage 在 Alpine/musl 跑不起来 |

---

## 关键环境变量

| 变量 | 说明 | 默认 |
|------|------|------|
| `BROWSER_USER` | 运行浏览器的用户。**容器里设 `root`**，见问题 3 | `browser` |
| `BROWSER_HOME` | 浏览器家目录 | `/home/$BROWSER_USER` |
| `BROWSER_WORK_DIR` | 工作区。**必须指向容器里 touch 得动的路径** | `$BROWSER_HOME/browser-work` |
| `BROWSER_USER_DATA_DIR` | 持久化 profile | `$BROWSER_WORK_DIR/persistent` |
| `BROWSER_XAUTHORITY` | X 授权文件 | `$BROWSER_HOME/.Xauthority` |
| `BROWSER_DISPLAY` | X 显示号 | `:1` |
| `PANEL_ROOT` | 面板根目录 | `/opt/browser-panel` |
| `PANEL_ENV` | 环境变量文件 | `$PANEL_ROOT/.env.panel` |

---

## proot 环境的其他坑

- `/` 是 **overlay 挂载且只读**，唯一可写的持久化路径是挂载卷（通常是 `/home/container`）
- **`chown` 有效但 `chmod` 是假的** —— 别相信 chmod 的返回码
- `whoami` / `id` 显示 root，**不代表有 root 权限**；proot 不真正切换 uid
- 装完先验一遍：

```bash
cd /opt/browser-panel && set -a && . ./.env.panel && set +a
node -e "console.log(require('./config').browser.workDir)"   # 应等于你设的路径
```

---

## 许可

沿用原项目许可。原始脚本版权归 [debbide/browser-panel](https://github.com/debbide/browser-panel) 所有。
