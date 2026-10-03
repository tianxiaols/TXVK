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

`<游戏根>\l4d2bridge\logs\bridge32.log` 开头应出现：

```
==================
L4D2 D3D9 Bridge Client
==================
Version: l4d2bridge-1.0.0+<hash>
```

同目录 `bridge64.log` 里应有 `L4D2 D3D9 Bridge Server`。**两个日志都含启动时的完整生效配置**，排障时请一并提供。

## 5. 常用开关（`bin\.l4d2bridge\bridge.conf`）

| 键 | 默认 | 说明 |
|---|---|---|
| `eliminateRedundantSetterCalls` | True | 状态去重（共 23 处：渲染状态/纹理/采样器/着色器/顶点流/渲染目标/着色器常量…）。怀疑状态类异常时关掉即可立刻排除 |
| `vkupShaderCache` / `vkupBufferPool` | True | 着色器缓存 / 顶点·索引缓冲池 |
| `vkupLockAudit` | True | 越界锁兜底（把越界写关进临时缓冲，保护游戏堆），建议保持开启 |
| `vkupShadowGuard` / `vkupShadowQuarantine` | False | 影子缓冲的守卫带 / 淘汰隔离校验；默认关闭以省 CPU，排障时可开 |
| `vkupStallReport` | True | 游戏长时间不出命令时打印**一次**停摆报告（含内存与最近命令） |
| `vkupClientDebug` | True | 每 5 秒一行 `[vkup] client stats`（命令数、去重命中、拷贝速率等），设 False 可完全安静 |
| `logLevel` | Info | 日志级别 |

## 6. 性能与诊断口径

- `bin\WOOL.log`（若系统里有 HookDLL 之类的 ODS 接收器）会包含我们的统计行：

  ```
  [vkup] client stats: fvf_skip=… shader_cache_hit=… pool_hit=… setter_skip=… | commands_total=… | locks=… outOfRange=0 scratchLocks=0 guardViol=0 qViol=0 … copy=xxxMB/s
  ```

  其中 `setter_skip` 是**被去重省掉的跨进程调用数**（实测约占全部调用的 12–15%），`copy` 是影子缓冲拷贝速率（实测 100–170 MB/s）。
- **诊断设计原则**：正常运行期**零周期扫描**，只在真的出事时留证据（见下节）。

## 7. 排障（出问题时请提供这些）

| 现象 | 取这三样 |
|---|---|
| 崩溃 | `<游戏根>\left4dead2.exe_<时间>.dmp` 与同名的 **`.vscript.txt`**（自动写出：出错模块+偏移、访问地址、故障线程 EBP 链、栈上的 `.nut` 名字）+ `%LOCALAPPDATA%\CrashDumps\` 下的 WER dump |
| 卡死/未响应 | `bin\.l4d2bridge\left4dead2_bridge_hang_*.dmp` + `l4d2bridge\logs\` 两个日志 + Windows 事件查看器里该时刻的 `Application Hang/Error` 记录 |
| 画面异常 | `l4d2bridge\logs\bridge32.log`（含启动配置）+ 是否关闭过 `eliminateRedundantSetterCalls` |
| 装不上/不生效 | `bin\WOOL.log` 里的加载链行（`[HookDLL] Loading …`）+ `bridge32.log` 开头 20 行 |

## 8. 已知限制

- 与真 **RTX Remix 运行时**或第三方 `l4d2-rtx` 包**不要混装**（它们使用 `.trex` 工作目录与 `NvRemixBridge.exe` 名字，本桥已刻意避开这些名字，但两套安装仍会互相干扰）。
- 本包不含 ENB / ReShade / HookDLL / 帧生成 / ncnn 模型；如需后处理请自行在 64 位侧解决。
- 文件属性里的版本资源显示 `FileVersion 1.0.0.0`，实际构建号见日志里的 `Version:` 行。

## 9. 许可与致谢

- 本项目自身：**MIT**（见 `LICENSE`）。
- 来源于并感谢：
  - **NVIDIA RTX Remix Bridge**（MIT，© NVIDIA CORPORATION）—— 32↔64 位桥的原始实现；
  - **DXVK**（zlib，© Philip Rebohle 及贡献者）—— 本包的 64 位 D3D9→Vulkan 实现；
  - **Microsoft Detours**（MIT）—— 32 位侧 API 拦截。

  完整许可见 `THIRD_PARTY.md`。
