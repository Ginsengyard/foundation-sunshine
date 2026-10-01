# mf-hwenc-fix 分支说明

这个分支在 `master` 基础上打了三处补丁，用于让**老 Intel 核显**（Ivy Bridge / HD 4000 及类似，
走 Media Foundation 的 `h264_mf`）真正用上硬件编码，并在客户端请求超出硬件能力时自动降级。

补丁细节、实测数据与完整调查报告见：
https://github.com/Ginsengyard/sunshine-encoder-patcher

## 三处改动

| 文件 | 改动 |
|---|---|
| `src/platform/windows/display_vram.cpp` | Intel 分支的能力白名单放行 `*_mf`（1 行），否则 `h264_mf` 在能力判定阶段就被丢弃 |
| `src/main.cpp` | 进程启动时保留一个 MF 平台引用（`MFStartup` 一次，不配对 `MFShutdown`），避免 ffmpeg 的 `mfenc` 每次打开都重建平台、丢失 MFT 的"打火"状态 |
| `src/video.cpp` | `*_mf` 编码器：请求宽或高超过 1920 时按比例钳制重试；最多尝试 6 次、失败之间间隔 250ms |

## CI 说明（fork 适配）

- `DRIVER_DEPS_REQUIRED` 改为与 RTX 变量相同的门控：只在官方仓库为 `ON`，fork 里为 `OFF`
- `Install Inno Setup` 改为 choco/winget 优先、官方直链兜底（原直链在 CI 环境偶尔被拒）
- 缓存键改为 `inno-setup-6-v2`

产物：Actions → Build and Release（手动触发，选本分支）→ `sunshine-windows-r<N>`，
内含 `Sunshine.*.WindowsInstaller.exe` 与便携包。

## 验证环境

ThinkPad E531（i7-3740QM + HD Graphics 4000，驱动 10.18.10.5161），Windows 10 22H2。
修复前：探测失败即回退 libx264；手机端 2400x1080 请求会连续失败（实测约 199 次）直到客户端断开。
修复后：720p / 1080p 走 `h264_mf` 出画面；2400x1080 自动降级为 1920x864 后正常出画面。