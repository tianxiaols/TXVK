# 第三方组件与许可 / Third-party components and licenses

TXVK 自身以 **MIT** 发布（见 `LICENSE`）。它建立在下列开源项目之上，保留其原始版权与许可：

| 组件 | 用途 | 许可 | 版权 |
|---|---|---|---|
| **NVIDIA RTX Remix Bridge** | 本桥的原始实现（32 位客户端 ↔ 64 位服务端的命令通道、D3D9 拦截框架） | MIT | © 2022-2023 NVIDIA CORPORATION |
| **DXVK** | `L4D2Bridge_d3d9_x64.dll` 是 DXVK 的 64 位 D3D9→Vulkan 构建 | zlib | © 2017-2025 Philip Rebohle and contributors |
| **Microsoft Detours** | 32 位侧的 API 钩子（`detours.cpp` 等随源码链接） | MIT | © Microsoft Corporation |

## NVIDIA RTX Remix Bridge (MIT)

```
Copyright (c) 2022-2023, NVIDIA CORPORATION. All rights reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## DXVK (zlib)

```
Copyright (c) 2017-2025 Philip Rebohle
Copyright (c) 2018-2025 DXVK contributors

This software is provided 'as-is', without any express or implied warranty.
In no event will the authors be held liable for any damages arising from the
use of this software.

Permission is granted to anyone to use this software for any purpose,
including commercial applications, and to alter it and redistribute it
freely, subject to the following restrictions:

1. The origin of this software must not be misrepresented; you must not claim
   that you wrote the original software. If you use this software in a
   product, an acknowledgment in the product documentation would be
   appreciated but is not required.
2. Altered source versions must be plainly marked as such, and must not be
   misrepresented as being the original software.
3. This notice may not be removed or altered from any source distribution.
```

## 未包含的第三方内容 / Not included

本发行包**不包含**任何以下内容，安装说明中也不会引导安装：

- ENBSeries（专有）
- ReShade / ShaderToggler
- 第三方 HookDLL / FrameGenPlugin / AIFrameService
- ncnn、RIFE 模型（帧生成相关）
- 游戏本体内容（VPK / 材质 / 模型 / 脚本）
- 任何 RTX Remix 运行时或 `l4d2-rtx` 项目文件

游戏本体、Steam 与 NVIDIA/AMD 驱动分别受各自许可约束。
