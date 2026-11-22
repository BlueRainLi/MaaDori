# 项目约定（AGENTS.md）

## 项目目标

- 本仓库自动化控制 **BanG Dream! 少女乐团派对（B 站国服）**。
- **App 包名：`com.bilibili.star.bili`**，可见 Activity 是 `.MainActivity`，
  但**启动入口是 `.PermissionActivity`**（`cmd package resolve-activity` 的解析结果）。
  （依据：`adb shell dumpsys window` 的 `mCurrentFocus`、`cmd package resolve-activity`，2026-11-23 实测）
- 包名落在 `resource/pipeline/startup.json` 的 `启动游戏` 节点（`action: StartApp`），
  由 `interface.json` 的 `启动游戏` 任务引用。

### StartApp 的坑（必须按组件形式写）

MaaFramework 的 `activity_.start_app(intent)` 按 intent 里**有没有 `/`** 分叉
（依据 `source/MaaAdbControlUnit/General/Activity.cpp` 的 `start_app` 与 `parse` 默认模板）：

| intent 形式 | 实际执行 |
| --- | --- |
| 不含 `/`（纯包名） | `adb -s <serial> shell monkey -p {INTENT} 1` |
| 含 `/`（包名/Activity） | `adb -s <serial> shell am start -n {INTENT}` |
| `StopApp`（任意） | `adb -s <serial> shell am force-stop {INTENT}` |

两条实测结论：

1. **本机 LDPlayer 镜像没有 monkey**：`adb shell monkey -p com.bilibili.star.bili 1`
   返回 `/system/bin/sh: monkey: inaccessible or not found`。所以**纯包名形式在本机会失败**，
   必须写成 `包名/Activity` 走 `am start`。
2. **`MainActivity` 未导出**：`am start -n com.bilibili.star.bili/com.bilibili.star.bili.MainActivity`
   报 `SecurityException: ... not exported`。启动入口要用
   `com.bilibili.star.bili/com.bilibili.star.bili.PermissionActivity`（实测可拉起）。

`StopApp` 的 `package` 直接填包名即可（`am force-stop com.bilibili.star.bili`）。

### 包名不要往 interface.json 里塞

PI v2 协议（`interface.json`）**没有** App 包名字段。MaaFramework 的启动/停止 App 是
Client 侧调 `MaaControllerPostStartApp(intent)` / `MaaControllerPostStopApp(intent)`，
Adb 控制器下 `intent` 传 package name 或 activity（依据 MaaFramework 集成接口文档
`2.2-集成接口一览.md` 的 `MaaControllerPostStartApp` / `MaaControllerPostStopApp`）。
→ 想让 MFAAvalonia 的「启动 App」按钮生效，包名要在 MFA 控制器面板里填；
写进 `interface.json` 只会当注释用，不生效。

## 本机调试环境

- **adb 路径：`D:\Tools\scrcpy-win64-v4.0\adb.exe`**（v37.0.0 / 1.0.41，随 scrcpy 附带）。
  系统 `PATH` 里没有 adb，`ANDROID_HOME` / `ANDROID_SDK_ROOT` 均未设置，
  Android SDK platform-tools 未安装。命令行用之前先：

  ```bash
  export PATH="/d/Tools/scrcpy-win64-v4.0:$PATH"
  ```

- 模拟器：LDPlayer 9（进程 `Ld9BoxHeadless.exe`），首个实例监听 `127.0.0.1:5555`。
  `adb devices` 里显示为 **`emulator-5554`**，与 `127.0.0.1:5555` 是同一台设备。
  `interface.json` 中 Adb 控制器的 `adb.address` 填的是 `127.0.0.1:5555`。
- 屏幕：1280×720，density 240。短边正好 720，符合 Maa 的 720p 资源基准，
  截图可以直接当模板图用，不需要缩放。

## Git Bash 下用 adb 的坑

MSYS 会把 `/sdcard/xxx` 当 Unix 路径改写成 Windows 绝对路径，设备端收不到参数，报：

```
Error opening file: <Git安装目录>/sdcard/screen.png (No such file or directory)
```

解决办法：设备路径写双斜杠，本地路径用相对形式。

```bash
adb shell screencap -p //sdcard/screen.png
adb pull //sdcard/screen.png ./screen.png
```

不要用 `MSYS_NO_PATHCONV=1` 绕过：它同时取消本地路径转换，`pull` 到 `/tmp/x.png`
会真的落到 `C:\tmp\`。

## 取坐标的方式

- **不要依赖 `uiautomator dump`**：本游戏是 Unity/SurfaceView，dump 出来整棵树只有
  5 个节点，最内层一个 `android.view.View`，`text` 全空，拿不到任何 bounds。
- 找坐标只能靠截图 + 图像识别（也就是 MaaFramework 自己的方式）。
  单次截图建议 `adb exec-out screencap -p > x.png`，省掉 `/sdcard` 往返。

## 发版（tag）规则

- 正式版 tag **一律三段式**：`v主版本.次版本.修订号`（如 `v0.1.0`），禁止两段式。
  原因：`install.yml` 的 changelog job 用 `--tag-pattern "^v[0-9]+\.[0-9]+\.[0-9]+$"`
  锚定正式版，两段式匹配不到任何标签，说明会整体落入「未发布」区块。
- 预发布可带后缀（`v0.1.0-beta.1`），meta job 会自动标记 prerelease。
- 构建矩阵**默认只有 `win-x86_64` 一个组合**（`install.yml` 的 `meta.set_matrix`）。
  需要全部 8 个平台组合时，手动触发 workflow_dispatch 并勾选 `full_build`，
  不要在 tag 发版路径上放开。

## 目录结构

- `interface.json`、`resource/` 在**仓库根目录**（已从模板的 `assets/` 铺平，与 MAADNA 一致）。
- `assets/` 下只保留 `MaaCommonAssets` 子模块（OCR 模型等），`.gitmodules` 的路径依赖它，
  不要挪动。本地没 init 时 `tools/configure.py` 会直接退出，跑 `install.py` 前先
  `git submodule update --init --recursive`。
- pipeline 节点：`resource/pipeline/<流程名>.json`。
- 打包：`tools/install.py <version> <os> <arch>`，产物在 `install/`。

## 待确认

- LDPlayer 的 `extras`（MaaFramework 的模拟器增强配置，用于加速截图/触控）未配置，
  需要时补 `adb.config.extras`，并把 LDPlayer 安装目录填进去。
- `interface.json` 的 `contact` 仍是模板占位「QQ群: 123456789」。
