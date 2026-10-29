# MiStep 从 HyperOS 4 (Android 17) 到 HyperOS 2.0 (Android 15) 适配计划

> **依据文档（地面真值，权威依据）：**
> 1. 《适配核对报告.md》（2026-10-26）：针对 HyperOS 2.0 真机 ROM（Android 15 / compileSdk 35）的 46 个目标类/方法逐项比对结果。
> 2. 《OneStep调研与替代映射.md》（2026-10-26）：路线确立为「做适配层替换、不重写；OS2 自带的 TaskView leash-reparent / setWindowBounds 栈完好，用它承接；不依赖锤子 Smartisan 框架层」。

---

## 一、适配路线与原则

1. **核心原则**：
   - 保持架构分工不变：SystemUI 宿主进程 UI + system_server 任务调度层。
   - 绝不在 OS2 上因类或方法缺失抛出未捕获异常，绝不崩溃或拖垮 SystemUI / system_server。
   - 对 OS2 完好的基础机制（ leashes、TaskView reparent、MagnificationSpec 缩放）直接复用。
   - 对 OS4 新增的边界/动画辅助 API 采用适配层替换或软化守卫降级。
   - 对 OS2 不存在的特性（灵动岛 DynamicIsland 等）增加运行时存在性守卫，安全跳过并打日志。

---

## 二、目标类与方法适配映射表

| 序号 | 目标类 / 方法 / API | OS2 状态（引核对报告） | 源码主要位置 | 处理方式 | 风险评估与应对 |
|:---:|---|---|---|:---:|---|
| 1 | `TaskViewTransitions.setTaskBounds` / `lambda$setTaskBounds$6` | ❌ 不存在（OS4 新增便捷方法） | `MiStepHost.java:1830-1831` | **适配层替换**（第一步加守卫，第二步改接 `TaskViewTaskController.setWindowBounds` + WCT） | 风险中：本轮先软化 hook，下一轮接上 setWindowBounds 实现副任务边界设置。 |
| 2 | `TaskViewTransitions.isValidTaskView` | ❌ 不存在（OS4 新增自检） | `MiStepHost.java:1833` | **运行时守卫停用**（移除 `throw IllegalStateException("no external task change guard")`，改为控制器自身状态判定） | 风险低：移除硬抛异常后，模块不会因检测不到该方法而自杀。 |
| 3 | `TaskViewTransitions.updateBoundsForUnfold` | ❌ 不存在（OS4 折叠屏展开方法） | `MiStepHost.java:1843` | **运行时守卫停用**（移除 `throw IllegalStateException("no unfold bounds guard")`，不存在时安全跳过） | 风险低：直板机（如 marble）上无需 unfold 边界动画。 |
| 4 | `TaskViewTransitions.prepareOpenAnimation` / `prepareCloseAnimation` | ❌ 不存在（OS4 新动画通道） | `MiStepHost.java:1795, 1863`<br>`MiStepLiveCollectTask.java:172`<br>`MiStepLiveTaskPair.java:263` | **适配层替换**（第一步移除硬异常守卫，第二步走 `startAnimation` 或 SurfaceControl 事务） | 风险中：本轮加存在性守卫防崩，下一轮补齐动画降级路径。 |
| 5 | `DynamicIslandGestureMonitor$listener$1` 及全部灵动岛类 | ❌ 缺失（OS2 无整个 `island` 包；对应物为 `notification.focus`） | `HostEntry.java:106`<br>`MiStepIslandMonitorInput.java:21`<br>`MiStepInputTrace.java:157, 159` | **运行时守卫停用**（检测到类不存在安全跳过并记录日志） | 风险极低：灵动岛为 OS4 独有，OS2 停用符合设计预期。 |
| 6 | `android.view.inspector.WindowInspector.addGlobalWindowViewsListener` / `removeGlobalWindowViewsListener` | ❌ Android 15 (API 35) 不存在（API 36/37 新增） | `MiStepStatusBar.java:97, 101, 160, 182` | **API level 守卫 + 反射降级**（在 API 35 上跳过 listener，回退到已有的静态 `getGlobalWindowViews()`） | 风险极低：解决编译缺失符号和运行时 NoSuchMethodError，平稳降级。 |
| 7 | `android.window.ScreenCaptureInternal`（及 `CaptureArgs$Builder`） | ❌ 缺失内部版；OS2 存在公开版 `android.window.ScreenCapture` | `SwapPreview.java:46, 51` | **适配层替换 / 兼容改指**（优先检测 Internal 版，回退到公开版 `ScreenCapture`） | 风险极低：公开版在 API 34/35 上签名稳定，`build()` 可用。 |
| 8 | `com.android.wm.shell.shared.handles.RegionSamplingHelper` | ⚠️ 包名变动：OS2 在 `com.android.systemui.shared.navigationbar.RegionSamplingHelper` | `MiStepStatusBarSampling.java:29` | **签名适配 / 兼容改指**（双向探测两个包名，存在即挂钩，均不存在则安全跳过） | 风险极低：确保状态栏亮暗色自适应采样在 OS2 上继续工作。 |
| 9 | `WindowState.getSystemUiVisibility` | ⚠️ Android 15 已删除该方法（可见性改为 Insets） | `HostEntry.java:2197` | **运行时守卫停用 / 适配替换**（`hookAllMethods` 找不到时返回空，不报错；后续转用 Insets） | 风险低：状态栏沉浸判断已主要走 Insets，不影响稳定性。 |
| 10 | `com.android.wm.shell.common.transition.MiuiTransitionHandlerUtil` | ❌ 不存在（OS4 新增 MIUI 过渡工具） | `HostEntry.java:2698` | **运行时守卫停用**（改用 `findClassIfExists`，不存在时安全跳过） | 风险极低：仅影响临时主区域裁剪动画，不影响核心逻辑。 |
| 11 | `com.android.wm.shell.common.transition.animation.MiuiAnimationContainer` | ❌ 不存在（OS4 新增过渡容器） | `MiStepInputTrace.java:191` | **运行时守卫停用**（捕获忽略，安全跳过） | 风险零：仅用于输入 trace 记录。 |
| 12 | `DisplayBackGestureHandlerImpl$inputEventReceiver$1` | ❌ 不存在（OS2 返回手势走 `EdgeBackGestureHandler`） | `HostEntry.java:2663` | **运行时守卫停用**（改用 `findClassIfExists`，第二阶段接线 EdgeBackGestureHandler） | 风险零：原代码仅用于返回手势诊断日志。 |
| 13 | `miui.systemui.controlcenter.*`（OS4 控制中心组件） | ❌ 缺失（OS2 控制中心结构不同） | `MiStepOs4Controls.java`<br>`MiStepControlCenter.java` | **运行时守卫停用 / 优雅降级**（未加载插件时磁贴与滑块降级显示占位，不崩溃） | 风险低：等待提取 `MiuiSystemUIPlugin.apk` 核实后决定能否改指。 |
| 14 | `com.miui.server.input.util.ShortCutActionsUtils` | ❓ 未知（待核对 `miui-services.jar`） | `MiStepCodes.java:43` | **运行时守卫停用**（已有 try-catch，缺失时返回 0 并打日志） | 风险零：扫码/乘车码快捷调用，有完整容错。 |
| 15 | Native `libmistep_raw.so`（libinputflinger 钩子） | ⚠️ 预置 17PM 画像，在 marble (OS2) 上跳过 | `NativeRawInputBridge.java`<br>`DeviceProfile.java` | **运行时守卫停用**（非 17PM 机型自动跳过，进入第二阶段） | 风险零：不影响启动与多任务逻辑，仅缺少原始输入坐标微调。 |
| 16 | 框架基础管道类（DisplayContent, Task, WindowState, Transition, SurfaceControl 等 36 个类） | ✅ 完全存在且签名一致 | `HostEntry.java`<br>`MiStepHost.java` 等 | **直接可用**（核心管道直接沿用） | 无风险。 |

---

## 三、构建环境缺口

1. **当前开发环境现状**：
   - **JDK**：已安装 OpenJDK 17 (`/home/hatch/jdk/bin/javac`) 与 OpenJDK 21 (`/home/hatch/workspace/rea/tools/jdk21/bin/javac`)。
   - **Android SDK**：只有 `android-35` 平台 (`/home/hatch/android-sdk/platforms/android-35/android.jar`)。
   - **Xposed API**：已存在 `libs/xposed-api-82.jar`。
   - **缺失项**：
     - 无 `android-37.0` 平台（Android 17 API）。
     - 无 Android NDK（NDK 28.2 等未配置在系统环境变量中，本轮不重新编译原生 .so）。
     - 无 Windows PowerShell 7（脚本为 `build.ps1` 与 `test.ps1`，Linux 环境无法直接执行 ps1 脚本）。

2. **验证策略与基准**：
   - 使用 Linux 原生命令进行静态语法与编译检查：
     ```bash
     javac -encoding UTF-8 -cp "/home/hatch/android-sdk/platforms/android-35/android.jar:libs/xposed-api-82.jar" -d /tmp/mistep-test-build $(find module/src/main/java -name "*.java")
     ```
   - **符号缺口区分规则**：
     - 若编译失败由 `WindowInspector.addGlobalWindowViewsListener` 引起，属于「源码硬依赖 API 37 符号」，必须通过 API level 守卫 + 反射修复，确保在 `android-35` 平台上能够零错误编译。
     - 若改动引入新的语法错误、导入缺失或类型不匹配，属于「改动本身写错」，严禁提交。

---

## 四、分阶段实施里程碑

- **第一阶段（本轮完成，高置信低风险改动）**：
  1. 修复 `MiStepStatusBar.java` 中对 API 37 特有 `WindowInspector` 监听器方法的直接依赖，增加 API level 守卫与反射降级，使整个工程在 `android-35` 下实现 **0 编译错误**。
  2. 软化 `MiStepHost.java` 中的 `installLogicalBoundsHooks` 硬校验：
     - 移除 `isValidTaskView` 缺失时抛出的 `IllegalStateException("no external task change guard")`；
     - 移除 `updateBoundsForUnfold` 缺失时抛出的 `IllegalStateException("no unfold bounds guard")`；
     - 移除 `prepareOpenAnimation` 缺失时抛出的 `IllegalStateException("no native opening guard")`；
     - 对 `setTaskBounds` 与 `lambda$setTaskBounds$6` 的查找增加安全容错，确保在 OS2 上不会因 NoSuchMethodError 中断 hook 安装流程。
  3. 强化灵动岛（`DynamicIsland`）组件的运行时存在性守卫：
     - 在 `MiStepIslandMonitorInput.java`、`HostEntry.java` 确保检测不到类时安全返回，杜绝潜在的崩溃隐患。
  4. 兼容重定向改指：
     - `MiStepStatusBarSampling.java`：增加对 OS2 包名 `com.android.systemui.shared.navigationbar.RegionSamplingHelper` 的兼容查找。
     - `SwapPreview.java`：增加对公开版 `android.window.ScreenCapture` 的兼容回退。
  5. 每次修改独立提交，清晰记录 commit 信息。

- **第二阶段（下一轮：TaskView 适配层替换与手势接线）**：
  1. 基于 OS2 完好的 `TaskViewTaskController.setWindowBounds` 与 WCT 实现副任务的尺寸设定与布局管理。
  2. 将 `prepareOpenAnimation` / `prepareCloseAnimation` 重定向为 OS2 的 `startAnimation` 或 SurfaceControl 事务。
  3. 将返回手势从缺失的 `DisplayBackGestureHandler` 改接至 OS2 的 `EdgeBackGestureHandler`。

- **第三阶段（补齐 ROM 材料后的细节迭代）**：
  1. 引入用户提取的 `MiuiSystemUIPlugin.apk`、`miui-services.jar`，核对控制中心组件与快捷方式。
  2. 针对目标机型（marble）提取 `libinputflinger.so` 符号，完成原生触摸坐标校准。
