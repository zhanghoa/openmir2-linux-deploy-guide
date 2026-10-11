# OpenMir2（传奇2私服）Linux 部署实战全记录

> 2026-10-11 实测完成于 Ubuntu 笔记本（192.168.0.41，aarch64/ARM），服务端六进程全部跑通。
> 引擎：[OpenMir2](https://github.com/mirbeta/OpenMir2)（C# .NET 8，GPL 系开源）+ [MirServer 基础数据](https://github.com/mirbeta/MirServer)。
> ⚠️ 法律边界：引擎开源合法；官方 Mir2 的美术/地图/音效资源为 Wemade 版权资产，无合法分发渠道，个人怀旧自用、责任自负。

## 一、架构与启动顺序

六进程，**必须按此顺序启动**（依赖关系决定）：

```
DBSrv(6000/5100/5700) → LoginSrv(5600/5500) → GameSrv(5000)
  → GameGate(7200) → SelGate(7100) → LoginGate(7000)
```

| 进程 | 作用 | 对外端口 |
|------|------|----------|
| DBSrv | 数据存储（MySQL 插件） | 6000（内部）/ 5100（选人网关用） |
| LoginSrv | 账号注册/登录 | 5600（内部）/ 5500（给 LoginGate） |
| GameSrv | 游戏引擎（M2Server 逻辑） | 5000（内部） |
| GameGate | 游戏网关（玩家数据转发） | **7200（客户端连）** |
| SelGate | 选人网关 | **7100（客户端连）** |
| LoginGate | 登录网关 | **7000（客户端连，入口）** |

- GameSrv 运行目录需含 `Envir/`（脚本/列表）、`Map/`（地图）、`server.conf`。
- `M2Server.dll`/`SystemModule.dll` 是 GameSrv 的依赖（NetChannel 代码），必须在 GameSrv 目录。

## 二、构建

```bash
git clone --depth 1 https://github.com/mirbeta/OpenMir2.git
cd OpenMir2 && dotnet publish src/GameSrv -c Release   # 各进程同理
```

- .NET 8 SDK。产出在 `src/<proc>/bin/Release/`（GameSrv 等核心进程无 `net8.0` 子目录；M2Server/WebApi 有）。
- DBSrv 用插件机制加载 `DBSrv.Storage.MySQL.dll`（需单独构建并放进 DBSrv 目录）。

## 三、踩坑清单（按致命程度排序，全是实测）

### 1. headless 崩溃：`Console.SetCursorPosition`（Linux 无 TTY）
`Maps.cs` 加载地图时擦行光标，无 TTY 时 `CursorTop-1 = -1` → `ArgumentOutOfRangeException` → 初始化中断。
**修复**：包一层 `if (!Console.IsOutputRedirected) { ... }`。

### 2. 初始化异常被吞 → 后续必崩 "Config为null"
`AppService.StartingAsync` 里 `_mirApp.Initialize()` 的 try/catch **不 rethrow**，只记"初始化游戏基础数据失败"。
后果：`NetChannel.Initialize()`（在 Initialize 之后）被跳过 → `TCPNetChannel.Start()` 抛
`ArgumentNullException: Config为null，请先执行Setup` → 进程 abort。
**排障要点：看到 "Config为null" 别查网络，先找前面被吞掉的 Initialize 异常。**

### 3. 缺失的 Envir 列表文件（开源数据不全）
代码 `GetEnvirFilePath("X.txt")` 用 `File.Open` 读，缺文件即 FileNotFoundException：
`DisableTakeOffList.txt / DisableMakeItem.txt / DisableSellOffItem.txt / EnableMakeItem.txt / GenMsg.txt / NoHptoExpMonList.txt`
→ 建空文件即可（空列表合法）。

### 4. sudo 建的文件属主是 root → ubuntu 进程写不进
`UnauthorizedAccessException: Access to the path ... GenMsg.txt is denied`。
**统一 `chown -R ubuntu:ubuntu run/ logs/`**。启动脚本以 ubuntu 身份运行（不要 sudo 嵌套 su，`su ubuntu` 要密码会失败）。

### 5. 配置大小写（Linux 敏感）
- GameSrv 读 **小写 `server.conf`**（`ConfConst.ServerFileName`），server-data 给的是大写 `Server.conf`。
- 必填 typo：`ConnctionString`（少个 N，别"修正"它）。
- **重建 run/ 会用 server-data 原版覆盖手工修正过的配置** → 所有本地化修正必须固化进重建脚本（大写小写两份都写）。

### 6. 授权白名单文件是作者内网 IP → "非法地址连接" 拒绝一切
- `LoginSrv/ServerAddr.txt`：允许连 LoginSrv 的服务器 IP 列表，原值 `10.10.0.x` → 改 `127.0.0.1`
- `DBSrv/AddrTable.txt`：允许连 DBSrv 的 IP → 改 `127.0.0.1`
- `DBSrv/ServerInfo.txt`：`IP IP 端口` 三列 → `127.0.0.1 127.0.0.1 7200`
**症状**：DBSrv↔LoginSrv 链接成功后立即"断开链接"死循环，LoginSrv 日志刷 `非法地址连接:127.0.0.1`。

### 7. Castle 路径 Windows 反斜杠（周期性异常）
代码默认 `CastleFile="Envir\\Castle\\List.txt"`，Linux 下 `\` 不是分隔符 → 每次在线数汇报/存盘报
`FileNotFoundException: .../Envir\Castle\List.txt`。
**修复**：Server.conf 加节（正斜杠，指向 server-data 自带的 Castle 数据目录）：
```ini
[Share]
CastleDir=Castle
CastleFile=Castle/List.txt
```
**大写小写两份 conf 都要加**（改完不重启不生效；初始化时读一次）。

### 8. DB 连接串
server-data 原版 `server=10.10.0.199;uid=root;pwd=123456` → 必改 `127.0.0.1;uid=root;pwd=;database=mir2;`。
MySQL 8.4 root 空密码 + `caching_sha2_password` 可用。三个库：`mir2`（角色）/ `mir2_account`（账号）/ `mir2_db`（数据），导入顺序：结构→账号→物品数据。

### 9. 残留进程占端口
重启前 `pkill -9 -f dotnet` 并确认端口释放，否则新进程绑端口失败 → 又是 "Config为null" 假象。

### 10. 正常现象（别误判为故障）
- 启动瞬间 `链接数据库/登录服务器失败` 一两条：后启动的进程还没就绪，10 秒内自动连上。
- GameSrv 周期性 `游戏网关[临时端口]已打开→已关闭`：在线数上报心跳（临时连接），非断链。
- `UserCastle.Initialize MainDoor.UnitObj = nil`：开源城堡数据缺城门对象，不影响运行。

## 四、运维脚本

**start-all.sh**（核心逻辑，ubuntu 身份运行）：
```bash
#!/usr/bin/env bash
export PATH=/usr/local/bin:$PATH
RUN=/home/ubuntu/openmir2/run; LOG=/home/ubuntu/openmir2/logs
PROCS=("DBSrv:DBSrv.dll" "LoginSrv:LoginSrv.dll" "GameSrv:GameSrv.dll" \
       "GameGate:GameGate.dll" "SelGate:SelGate.dll" "LoginGate:LoginGate.dll")
for p in "${PROCS[@]}"; do
  IFS=':' read -r name dll <<< "$p"
  pgrep -f "$dll" >/dev/null && { echo "[$name] 已运行"; continue; }
  cd "$RUN/$name" && dotnet "$dll" > "$LOG/$name.log" 2>&1 &
  echo $! > "$LOG/$name.pid"; sleep 4
done
```
stop-all.sh 逆序 kill；status.sh 检查 6 进程 + 6 端口（7000/7100/7200/5000/5600/6000）。

**验证健康标准**：6 进程存活、6 端口 OPEN、`ss -tn` 互联 ESTABLISHED ≥ 8 条、日志无"非法地址"/无 Unhandled。

## 五、客户端（最难的最后一公里）

- 官方 [mirbeta/MirClient](https://github.com/mirbeta/MirClient)（Delphi 源码，`main` 分支，工程 `Source\MirClient\MirClinet.dproj`，需 Delphi XE 10.4 + DelphiX 组件[仓库 Components/ 内]）。**官方无编译版**——原发布群因版权"被抓"已解散（见官方 Issue #1）。
- 社区标准做法：**原版热血传奇 1.76 客户端** + 改 `Mir2.ini`（`ServerAddr=192.168.0.41` / `ServerPort=7000`）+ `ServerList.txt`（`名称|IP|7000|0|1.76|0|0`）。协议兼容已被社区实测（fork 作者 gqf2008 真机跑通登录→选人→进图）。
- 客户端所需资源（Prguse.data/Hum.data/Weapon.data/Tiles.data 等）**无合法下载源**，只能从自有老客户端提取。

## 六、替代方案：Zircon 传奇三（如不执念 1.76）

[zircon-legend](https://github.com/raphaelcheung/zircon-legend-server)（AGPL-3.0，297★，2026-10 仍在更新）：
- 传奇3（4 职业/38 技能），服务端有 Docker 镜像 `raphzhang/zirconlegend:latest`，客户端官方 Releases 带 exe + 网盘 3GB 资源包（作者分发，开箱即玩，资源授权灰色同上）
- **无网页版**：7080 端口是 Web 管理后台，不是网页客户端；客户端是 Windows 桌面程序（.NET Fx 4.8 + DirectX 9）
- 与 Mir2 客户端互不通用

## 七、本机部署资产清单（已删除前留档）

```
/home/ubuntu/openmir2/
├── git/            # OpenMir2 源码（含 Maps.cs headless 补丁）
├── server-data/    # mirbeta/MirServer 数据（已本地化修正）
├── run/            # 六进程运行目录（构建产物+配置+数据）
├── build-plugins/  # DBSrv MySQL 存储插件
├── start-all.sh / stop-all.sh / status.sh
└── logs/
MySQL: mir2 / mir2_account / mir2_db 三库
```
本地脚本副本：`rebuild_run.py`（一键重建+固化全部修正）、`rewrite_conf.py`（六进程配置模板）。
