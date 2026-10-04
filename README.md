# TXVK —— L4D2 的 32→64 位 D3D9 桥（Vulkan / DXVK）

把 32 位《Left 4 Dead 2》进程里的每一条 D3D9 调用送到 **64 位**进程、由 **DXVK（Vulkan）**真正执行，
从而绕开 32 位地址空间与老旧 D3D9 驱动的限制，并顺带修掉几条引擎自身的崩溃/刷屏路径。

> English TL;DR: TXVK bridges every D3D9 call of the 32-bit Left 4 Dead 2 process to a 64-bit
> process that renders it with DXVK (Vulkan). Ships a 32-bit client shim, a 64-bit server and a
> 64-bit DXVK build. No ENB / ReShade / third-party plugin is bundled.

---

## 1. 它是什么 / 不是什么

- **是**：一个可插拔的 D3D9 桥 + 一份 64 位 DXVK。装完即用，配置全在 `bin\.l4d2bridge\`。
- **不是**：不是 RTX Remix（不启用 Remix 运行时/光追），不是 ENB/ReShade 的替代品，
  也不包含任何第三方滤镜、帧生成或 HUD 插件。

## 2. 环境要求

| 项 | 要求 |
|---|---|
| 系统 | Windows 10 / 11 x64 |
| 显卡驱动 | **支持 Vulkan**（必需） |
| 游戏 | Left 4 Dead 2（Steam） |
| 运行库 | 无需额外安装：本包三个二进制都是静态 CRT 或只用系统 DLL |

## 3. 安装（3 步）

1. **备份**你现有的 `...\Left 4 Dead 2\bin\dxvk_d3d9.dll`（若装过 DXVK 或 ENB 代理，这就是它）。
2. 把本包 `bin\` 覆盖进游戏目录（保持目录结构）：

   ```
   Left 4 Dead 2\
   └─ bin\
      ├─ dxvk_d3d9.dll                  ← 32 位客户端（引擎按这个名字加载）
      └─ .l4d2bridge\
         ├─ L4D2Bridge64.exe            ← 64 位服务端（客户端自动拉起）
         ├─ L4D2Bridge_d3d9_x64.dll     ← 64 位 DXVK
         └─ bridge.conf / dxvk.conf / vkup.cfg
   ```
3. Steam → 库 → Left 4 Dead 2 → 属性 → **启动选项**：

   ```
   -vulkan -insecure
   ```

建议在游戏内使用**窗口化 / 无边框**（显示模式由 64 位侧接管，窗口化最稳）。

> 若你的加载链里还有 ENB / HookDLL（它们按 `dxvk_d3d9_last.dll` 这个名字加载下一环），
> 请把 `bin\dxvk_d3d9.dll` 再复制一份、改名为 `bin\dxvk_d3d9_last.dll`。

## 4. 验证是否生效

跑一次游戏（进图 → 玩一会 → 退出），然后看两处。

### 4.1 成功标记（最直观）

```
Left 4 Dead 2\bin\.l4d2bridge\RUN_OK.txt
```

这个文件**每次启动游戏都会被删除**，只有当桥真的跑起来（启动握手完成、已有 D3D9 调用被送到 64 位侧）才会重新写出。所以它的有无本身就是结论：

| 现象 | 含义 |
|---|---|
| 文件在，`state : clean shutdown …` | 本次运行完全正常（含干净退出）✅ |
| 文件在，`state : running` | 桥跑起来了，但进程没有干净退出（崩溃/强杀）——去看 `errors\` ⚠️ |
| 文件**不在** | 这次根本没跑到"能渲染"的状态；原因在 `l4d2bridge\errors\` 与两份日志里 ❌ |

文件内容含版本号、会话时长、转发命令数、退出原因，以及**要提供给作者的目录路径**。

### 4.2 日志开头

`<游戏根>\l4d2bridge\logs\bridge32.log` 开头应出现：

```
==================
L4D2 D3D9 Bridge Client
==================
Version: l4d2bridge-1.0.0+<hash>
```

同目录 `bridge64.log` 里应有 `L4D2 D3D9 Bridge Server`。**两个日志都含启动时的完整生效配置**，排障时请一并提供。

### 4.3 运行目录（首次运行自动生成）

```
Left 4 Dead 2\
├─ bin\.l4d2bridge\                 ← 程序与配置
│  ├─ L4D2Bridge64.exe
│  ├─ L4D2Bridge_d3d9_x64.dll
│  ├─ bridge.conf / dxvk.conf / vkup.cfg
│  └─ RUN_OK.txt                    ← 「上次运行成功」标记（每次启动重写）
└─ l4d2bridge\
   ├─ logs\                         ← 正常日志：都发生了什么
   │  ├─ bridge32.log  bridge64.log     客户端 / 服务端完整日志（含启动配置）
   │  ├─ run_history.log                每局的 START / OK / EXIT 各一行
   │  └─ d3d9.log                       64 位 DXVK 自己的日志
   └─ errors\                       ← 只有"出问题"的东西：哪里出了问题
      ├─ bridge32.errors.log  bridge64.errors.log   仅 warn / err 行（不必翻全量日志）
      ├─ <进程名>_crash_<时间>.dmp                 崩溃 minidump
      ├─ <进程名>_bridge_hang_<pid>.dmp            卡死时看门狗写的全线程 dump
      └─ <dump 同名>.vscript.txt                   崩溃时的脚本(.nut)轨迹
```

`<进程名>` 是 `left4dead2`（游戏进程，32 位侧）或 `L4D2Bridge64`（服务端进程）——谁出的事一眼可见。

> **要让别人给你排障时，只需打包这三样**：`l4d2bridge\logs\` 整个目录、`l4d2bridge\errors\` 整个目录、
> `bin\.l4d2bridge\RUN_OK.txt`。九成问题靠这三样即可定位，不必再让对方翻游戏目录。

## 5. 常用开关（`bin\.l4d2bridge\bridge.conf`）

| 键 | 默认 | 说明 |
|---|---|---|
| `eliminateRedundantSetterCalls` | True | 状态去重（共 23 处：渲染状态/纹理/采样器/着色器/顶点流/渲染目标/着色器常量…）。怀疑状态类异常时关掉即可立刻排除 |
| `vkupShaderCache` / `vkupBufferPool` | True | 着色器缓存 / 顶点·索引缓冲池 |
| `vkupLockAudit` | True | 越界锁兜底（把越界写关进临时缓冲，保护游戏堆），建议保持开启 |
| `vkupShadowGuard` / `vkupShadowQuarantine` | False | 影子缓冲的守卫带 / 淘汰隔离校验；默认关闭以省 CPU，排障时可开 |
| `vkupStallReport` | True | 游戏长时间不出命令时打印**一次**停摆报告（含内存与最近命令） |
| `vkupClientDebug` | True | 每 5 秒一行 `[vkup] client stats`（命令数、去重命中、拷贝速率等），设 False 可完全安静 |
| `vkupServerVkSanitize` | True | 给 64 位服务端一份干净的 Vulkan 环境：不加载第三方注入层/游戏助手/覆盖层层，并跳过微软 D3D Mapping Layers（Dozen）这个多余的 Vulkan-on-D3D12 后端。这些层与后端只会拖慢服务端、增加崩溃面。要保留某些层用 `vkupServerVkKeep`（默认 `reshade`）；完全关闭本功能设 False |
| `vkupDxgiGuard` | True | 拦截对**游戏目录里那份 DXVK `dxgi.dll`**（以及本地 `d3d10/d3d11/d3d12`）的加载，改为加载系统副本。桥下 32 位进程不渲染，而 Steam 等组件在过图时会去调 `CreateDXGIFactory`，那份本地 dxgi 会就地初始化 Vulkan 并死锁 → 表现就是"进图卡住"。**本项默认开启，因此无需删除任何文件**；日志里 `[dxgi-guard]` 会记录是谁在加载它 |
| `vkupSteamInvite` | True | 内置"Steam 好友邀请"窗口（替代画不出来的 Steam 覆盖层）：游戏里点"邀请好友"即弹出（兜底热键 **Ctrl+Alt+I**）。列好友 → 「邀请到我的大厅」用 `InviteUserToLobby`、「加入他的游戏」用好友的 lobby。邀请前需已在游戏大厅里，窗口底部会显示当前大厅。关闭设 False |
| `logLevel` | Info | 日志级别 |

## 6. 性能与诊断口径

- `bin\WOOL.log`（若系统里有 HookDLL 之类的 ODS 接收器）会包含我们的统计行：

  ```
  [vkup] client stats: fvf_skip=… shader_cache_hit=… pool_hit=… setter_skip=… | commands_total=… | locks=… outOfRange=0 scratchLocks=0 guardViol=0 qViol=0 … copy=xxxMB/s
  ```

  其中 `setter_skip` 是**被去重省掉的跨进程调用数**（实测约占全部调用的 12–15%），`copy` 是影子缓冲拷贝速率（实测 100–170 MB/s）。
- **诊断设计原则**：正常运行期**零周期扫描**，只在真的出事时留证据（见下节）。

## 7. 排障（出问题时请提供这些）

| 现象 | 取这些 |
|---|---|
| 崩溃 | `l4d2bridge\errors\<进程名>_crash_<时间>.dmp` 与同名的 **`.vscript.txt`**（自动写出：出错模块+偏移、访问地址、故障线程 EBP 链、栈上的 `.nut` 名字）+ `errors\bridge32.errors.log` + `%LOCALAPPDATA%\CrashDumps\` 下的 WER dump |
| 卡死/未响应 | `l4d2bridge\errors\<进程名>_bridge_hang_<pid>.dmp`（`left4dead2_…` = 游戏进程，`L4D2Bridge64_…` = 服务端）+ `logs\` 两份完整日志 + Windows 事件查看器里该时刻的 `Application Hang/Error` 记录 |
| 进图卡住（黑屏/卡在载入条） | 先看 `logs\bridge32.log` 里有没有 `[dxgi-guard]` 行（记录是谁请求了本地 dxgi）与 `[vk-sanitize]` 行；若卡死反复出现，把 `l4d2bridge\errors\left4dead2_bridge_hang_*.dmp` + `bridge32.log` 一起提供 |
| 改视频选项后卡死 | 同上；另外在 `logs\bridge32.log` 里搜 `Reset():` —— 只有 `begin` 没有 `done in … ms` 说明卡在设备重置里，把这两行连同上面的 dump 一起提供即可定位 |
| 画面异常 | `l4d2bridge\logs\bridge32.log`（含启动配置）+ 是否关闭过 `eliminateRedundantSetterCalls` |
| 装不上/不生效 | 先看 `bin\.l4d2bridge\RUN_OK.txt` 在不在（不在 = 从未跑起来），再看 `errors\bridge32.errors.log` 与 `bridge32.log` 开头 20 行；若系统里有 HookDLL 之类的 ODS 接收器，`bin\WOOL.log` 里的加载链行也有用 |

## 8. 已知限制

- 与真 **RTX Remix 运行时**或第三方 `l4d2-rtx` 包**不要混装**（它们使用 `.trex` 工作目录与 `NvRemixBridge.exe` 名字，本桥已刻意避开这些名字，但两套安装仍会互相干扰）。
- 本包不含 ENB / ReShade / HookDLL / 帧生成 / ncnn 模型；如需后处理请自行在 64 位侧解决。
- 游戏根目录残留的 `dxgi.dll`（旧 DXVK 留下）**不需要手工删除**：本桥的 dxgi 守卫会拦截对它的加载，改加载系统副本（日志里 `[dxgi-guard] … from <谁> -> C:\Windows\SysWOW64\DXGI.dll`）。历史故障（Steam 抓帧在过图时调 `CreateDXGIFactory` 撞上本地 dxgi → 死锁 → 卡在加载页）由此根除。
- Steam 覆盖层本身（Shift+Tab 界面、截图、FPS 显示）在桥下不可用：画面在 64 位服务端渲染，Steam 注入的是 32 位游戏进程，覆盖层拿不到可绘制的设备。
  **但「邀请好友」已内置**：游戏里点「邀请好友」（或按 **Ctrl+Alt+I**）会弹出本桥自己的好友窗口 —— 列表按「在线/是否在 L4D2 房间」标注，「邀请到我的大厅」直接调 Steamworks（对方 Steam 会弹提示），「加入他的游戏」直接进对方的房间；窗口底部显示当前大厅，邀请前请先在游戏里建房或进一局。
  本桥已把 `ISteamUtils::IsOverlayEnabled()` 对游戏报为 `true`，所以游戏不会再弹「此功能需要开启游戏中的 Steam 社区」。**你不需要去 Steam 里开启覆盖层**（开着反而会重新引入 Steam 抓帧去加载本地 dxgi 的老路径）。若想关掉整个邀请窗口功能：`vkupSteamInvite = False`。
- 文件属性里的版本资源显示 `FileVersion 1.0.0.0`，实际构建号见日志里的 `Version:` 行。

## 9. 许可与致谢

- 本项目自身：**闭源**（暂不提供源码，后续可能开源；见 `LICENSE`）。
- 来源于并感谢：
  - **NVIDIA RTX Remix Bridge**（MIT，© NVIDIA CORPORATION）—— 32↔64 位桥的原始实现；
  - **DXVK**（zlib，© Philip Rebohle 及贡献者）—— 本包的 64 位 D3D9→Vulkan 实现；
  - **Microsoft Detours**（MIT）—— 32 位侧 API 拦截。

  完整许可见 `THIRD_PARTY.md`。
