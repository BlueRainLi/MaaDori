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

## 启动流程（pipeline 链）

`resource/pipeline/startup.json` 目前是三节点链：

```
启动游戏 (StartApp) ──next──> 闪屏加速 (DirectHit + Click ×9) ──next──> 进入游戏 (TemplateMatch + Click)
```

- **`启动游戏`**：`StartApp` 拉起 App，`post_delay: 1500` 后进下一节点（别对着黑屏点）。
- **`闪屏加速`**：闪屏阶段点画面能跳开场动画，标题画面提前约 10s 出现。
  `DirectHit` 无条件命中 + `repeat: 9` / `repeat_delay: 900` 连点屏幕正中。
  实测（冷启动 force-stop 后重开，逐秒截图统计标题按钮首次 ≥0.8 的时刻）：

  | 点击情况 | 标题出现 |
  | --- | --- |
  | 不点 | 19.4s |
  | 5 下（1.5–5.5s） | 16.6s |
  | 5 下（1.5–9.8s） | 11.1s |
  | 8 下（1–8.7s） | 9.6s |
  | 9 下（1.5–8.9s） | **9.9s** |
  | 12 下（1–10.6s） | 10.9s |

  → 约 9–10 下、间隔 1s、覆盖到第 9 秒左右最好，再多没有额外收益（剩余时间在等游戏自身加载）。
- **`进入游戏`**：以标题画面**右下角「四宫格图标 + 菜单」按钮**作为「标题画面已就绪」的锚点，
  识别到后点屏幕正中 `[640, 360]`（触发 `TAP TO START`）。
  - 模板：`resource/image/title_menu_button.png`，64×73，从 1280×720 原生截图
    `(1190, 627)` 处裁剪，与 720p 基线 1:1。
  - `roi: [1140, 600, 140, 120]` 限制在右下角——左下角还有一个「logo + 菜单」按钮，
    不限 roi 会混淆（实测左下角得分 0.42，真按钮 1.0，主界面 0.23，阈值 0.8 余量充足）。
  - `timeout: 120000` 兜底，正常 10s 内就出标题。
- **注意时序**：点完 `TAP TO START` 到真正进主界面约 **15–18s**（要过加载条）。
  后续若接主界面节点，该节点的 `timeout` 要自己留够，不要指望 `post_delay`。

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

## 弹窗 / 教程拦截机制（`[JumpBack]`）

游戏运行中随时可能弹出教程引导（按钮文案先是「下一步」，翻到最后变成「关闭」），
它会盖住真正的业务界面。处理方式是 **MaaFramework 的 `[JumpBack]` 跳回机制**
（协议文档 `3.1-任务流水线协议.md` 的「中断机制」，文档里给的应用场景就是
「网络断开提示框、权限请求弹窗等临时性界面」）。

拦截节点在 `resource/pipeline/common.json` 的 **`教程弹窗`**：

```jsonc
"教程弹窗": {
    "recognition": "OCR",
    "roi": [400, 540, 880, 180],       // 右下区域，别全屏 OCR
    "expected": ["下一步", "关闭"],     // 文案会变，两个都接
    "threshold": 0.5,
    "action": "Click",
    "target": true,                     // 点识别到的文字框中心
    "post_delay": 700,                  // 等翻页动画
    "rate_limit": 500
}
```

**用法：需要防弹窗的业务节点，把 `"[JumpBack]教程弹窗"` 追加到自己 `next` 的末尾。**

```jsonc
"某业务节点": {
    "next": ["真正的目标节点", "[JumpBack]教程弹窗"]
}
```

执行逻辑：父节点每轮按顺序识别 `next` → 目标没出现就先命中拦截节点 → 点一下「下一步」
→ 该节点链执行完毕后自动**跳回父节点**重新识别。教程有多页也没问题，点一次跳回一次，
直到弹窗消失、真正的目标能被识别到为止。放在末尾保证业务目标优先级更高。

**注意**：
- 拦截节点自己**不要写 `next`**。写了自循环就变成「一直点不回来」；空 `next` 才能
  让节点链正常结束并触发回跳。
- 每多挂一个 `[JumpBack]` 节点，父节点每轮识别就多一次 OCR 开销，只挂真正需要的。
- 依赖 OCR 模型：`resource/model/ocr/` 下要有 `det.onnx` / `rec.onnx` / `keys.txt`
  （本地已配好，30MB，被 `resource/model/.gitignore` 忽略，不进仓库）。

## 业务流程：抽每日3抽（`resource/pipeline/daily_free_roll.json`）

```
主界面(And 三按钮) → 点击招募 → 招募页(TemplateMatch 招募详情) → 招募列表上滑(Swipe ×3) → 每日免费3抽(定位)
```

`interface.json` 的任务「抽每日3抽」入口是 `主界面`。

- **`主界面`**：`And` 识别底部按钮条（roi `[600,580,660,130]`）里 **招募 + 乐队 + 故事** 三个模板
  全部命中才算主界面。三个按钮互相之间的误匹配最高 0.63，各自本尊 1.0，阈值 0.8 安全。
- **`点击招募`**：模板命中框中心即按钮位置 `(695, 647)`，`target: true` 直接点。
- **`招募页`**：锚点是底部常驻的「招募详情」文字，模板只裁文字（90×30）。
  ⚠️ **不要裁整个白色胶囊**：胶囊是半透明白底会透出卡面，换个活动页得分就从 1.0 掉到 0.85；
  → 纯文字模板在 STEP UP / 少女们的回忆 / 指挥官祭 三个页面都是 1.000，主界面 0.278。
- **`招募列表上滑`**：`Swipe` 从 `(130,520)` 拖到 `(130,140)`，350ms，`repeat: 3`。
  左侧列表的**最后一项就是「每日3次免费！演出招募」**（实测滑到底后第 3、4 次上滑画面完全不变，
  说明列表是线性的、到底会夹住，多点几下不会滑过头）。从列表顶部滑 3 次即可到底。
- **`每日免费3抽`**：模板 `roll_daily_free_card.png`（145×65，只取卡面文字不取红色选中边框）。
  实测选中 1.000、未选中 0.81、不在视野 0.39，阈值取 **0.75**。
  节点只定位不点击，下一步（点这张卡开始抽）待定。

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
  不要挪动。**本机已 init（431MB，commit `2327247`）**，`resource/model/ocr` 也已由
  `python tools/configure.py` 拷好；换机器/新 clone 后仍需
  `git submodule update --init --recursive` 再跑一次 `configure.py`。
- pipeline 节点：`resource/pipeline/<流程名>.json`，公共拦截节点放 `common.json`。
- 打包：`tools/install.py <version> <os> <arch>`，产物在 `install/`。

## 待确认

- LDPlayer 的 `extras`（MaaFramework 的模拟器增强配置，用于加速截图/触控）未配置，
  需要时补 `adb.config.extras`，并把 LDPlayer 安装目录填进去。
- 主界面（`CHU²的公寓`）还没有识别节点，`进入游戏` 的后续流程待补。
