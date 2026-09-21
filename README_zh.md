# MiFitnessAdAway

[English](README.md) | 中文

小米运动健康（`com.mi.health`，含国际版 `com.xiaomi.wearable`）去广告 LSPosed 模块，基于现代 **libxposed API 102** 开发。

> **v1.3.0，真机验证通过** —— Redmi K70 至尊版 / Android 16 / 运动健康 3.59.1（早期版本为一加 PLQ110）：下列各页均保持干净，正常功能完好，试用表盘可导出供第三方导入，手环 ↔ 手机勿扰同步恢复正常。

## 去掉了什么

分组方式与设置页卡片一致，按「它属于哪个界面」找开关：

| 位置 | 清除内容 |
|---|---|
| 开屏 | 开屏广告（图 / 视频） |
| 弹窗 | 「发现新版本」更新弹窗 · 全屏会员推广弹窗 |
| 首页（健康） | 健康页推广卡片 |
| 设备页 | 推广卡片 · 底部「设备」tab 红点 · 首页「系统设置」入口红点 |
| 运动页 | 轮播卡片 · 「训练指标」以下整个运营区（并禁用页面滚动） |
| 我的页 | VIP 会员卡 · 健康问诊卡 |
| 健康详情页 | 平安健康问诊卡（睡眠 / 心率 / 血氧 / 压力）· 蚂蚁阿福 AI 解读卡 · 睡眠研究卡与 21 天改善卡 |
| 体重页 | 个性化减重方案栏 |
| 全局 | 小米验证 SDK 的 hook 检测（`SensorHelper`） |

## 设置界面

模块内一个界面、17 个开关，无需额外配置。

| 卡片 | 开关 |
|---|---|
| 总开关 | 去广告总开关 —— 单独一张卡，与「已启用 x/11」摘要同卡显示 |
| 开屏与弹窗 | 开屏广告 · 应用更新弹窗 · 会员推广弹窗 |
| 我的页 | VIP 会员卡 · 健康问诊卡 |
| 运动页 | 轮播卡片 · 运营卡片（训练指标以下） |
| 设备页 | 红点（底部 tab + 系统设置入口）· 勿扰同步（手机 ↔ 手环） |
| 健康详情页 | 问诊卡片（睡眠 / 心率 / 血氧 / 压力）· 睡眠研究/改善卡片 · 减重方案栏 |
| 表盘 | 试用表盘自动导出（实验，默认关闭） |
| 其他 | 反 hook 检测 · 调试日志 · **隐藏桌面图标**（即时生效） |

- 点分组标题即可收起 / 展开，折叠状态会记住。
- 改完开关需**重启运动健康**生效，无需重启手机。
- 总开关关闭时，依赖它的开关一律变灰不可点；「调试日志」「隐藏桌面图标」「勿扰同步」不受总开关约束，保持可用。
- 界面颜色跟随系统深色 / 浅色模式。

## 附加功能

- **试用表盘自动导出（实验）** —— 试用下载完成后，缓存目录哨兵立刻把包体读走：后台线程每 400ms 轻量看一眼 `WatchFace/<设备>/<表盘ID>/`（只 stat、不读盘），发现新包体、等它 300ms 落稳后当场读出来（试用包体在这个目录里只停很短时间——实机实测 32 位 MD5 命名的包体出现后 1 分钟内就被 App 删掉；推送入口 `doInstall`/`preInstall` 在小米手环 8 Pro 上根本不被调用，只靠 hook 或「开我的页扫一次」都会踩空）。包体名字对不上时按包体头部里的表盘 ID 兜底匹配：老版 App 固定命名为 `resource.bin`，新版 App 以内容 MD5 命名（32 位十六进制、无扩展名）。定位到包体后按「12→19」等长换新 ID，以表盘中文名写入（名字取自缓存目录的 `description.xml`；新版缓存没有该文件时，改从包体头部的名称字段读取），供第三方软件导入。导出位置可在 设置 → 表盘 → 「导出位置」里自定义：填相对路径，如 `Download/表盘导出`、`Documents/2026/面孔`（首级目录仅限 `Download` / `Documents`，这是 MediaStore 对第三方 App 的全部放行范围，其他会回退 `Download`）。改位置时会问是否把已导出的表盘重导过去，且去重按位置判定，保证新目录真的能收到文件。导出 ID 会从服务端清理名单中摘除，第三方刷入的表盘同步不再被删；已导出缓存下次扫描即清理（快照 + 交接/推送保护）。每次扫描经 Toast/通知告知结果（含真实落盘位置）。
- **手环 ↔ 手机勿扰同步（Android 15+）** —— 官方同步在 Android 15 及以后是死的：`ZenUtils.isSupportZenMode()` 在那里无条件返回 `false`，而整条链路都被它门控。打开本开关后：放行该判断、App 启动后重新挂上 `zen_mode` 内容观察者、同步开关保持开启、并在启动时拉取一次手环侧规则。需要给运动健康授予「勿扰访问权限」——该权限由系统校验，模块只能提示。
- **隐藏桌面图标** —— 即时隐藏模块自身图标，仍可从 LSPosed 打开设置页。
- **调试日志** —— 输出详细 hook 日志，便于排查。

## 实现原理

- **数据层优先拦截**：banner 接口、开屏缓存、会员数据、问诊数据、平安健康 banner 绑定方法直接返回空或跳过，广告在渲染前就被掐掉。
- **视图层兜底**：若干页面是 React Native（YRN）渲染。这类页卡共用 RN 宿主 `YRNCFragment`，通过扫描视图树、按标题文本定位后整卡移除 —— 这些页面文案由服务端下发，没有稳定的数据接口可钩。
- **可靠性处理**：「我的」页逐层上卷隐藏卡片并上移后续内容；运动页按「训练指标」锚点整体移除以下区域；已处理的行会记账，反复扫描不会重复上移。
- **设备红点**：伪装 `PowerManager.isIgnoringBatteryOptimizations` 返回 true（等效「已忽略电池优化」），并让表盘红点 getter 返回 false，一举消除底部 tab 红点与首页「系统设置」入口红点。
- **弹窗拦截**：更新弹窗跳过 `AppUpgradeUtil.showUpdateDialogIfNeed`；会员推广弹窗跳过 `MembershipDialogManager.showMembershipExpiredFaceDialog`，同时仍回调调用方的 dismiss 回调，保证它继续拉取的生日勋章流程不受影响；用户主动点「开通会员」的购买弹窗不受影响。
- **版本容错**：每个 hook 独立安装、失败隔离，某版本入口有差异时自动跳过，其余照常生效。
- **延迟初始化类**：静态初始化依赖 Application 上下文的类（`DeviceSettingsPreference`、`ZenUtils`、`FitnessApp`）一律用 `initialize = false` 加载 —— 安装期强制触发 `<clinit>` 会抛异常，hook 会直接装不上。

## 要求

- 真机 root（KernelSU 或 Magisk）+ LSPosed ≥ 2.1.1（Zygisk）
- 小米运动健康 `com.mi.health` / `com.xiaomi.wearable` 3.0+

## 安装

1. 从 [Releases](../../releases) 下载 `MiFitnessAdAway-*.apk`。
2. 在 LSPosed 中启用模块（静态作用域已含两个包名）。
3. 重启一次，打开模块图标（或 LSPosed → 模块设置）调整开关。

## 构建

需要 Gradle 9.5.1、AGP 9.2.1、JDK 17、compileSdk 37。

```powershell
gradle assembleRelease   # 产物 app/build/outputs/apk/release/app-release.apk
```

本地存在 `keystore/mifitnessadaway.keystore` 与 `keystore/signing.properties`（均不入库）时使用正式签名，否则回退 debug 签名。GitHub Actions 会在每次 push / PR 自动构建，但该产物是 debug 签名，只用于验证能否编译通过。

## 仓库结构

```
app/src/main/java/io/github/hao1196561270/mifitnessadaway/
├── AdAwayModule.java     # libxposed 入口（全部 hook）
├── SettingsActivity.java # 设置界面（分组卡片、深浅色自适应）
├── MiFitnessApp.java     # XposedService 桥接（RemotePreferences）
└── Prefs.java            # 设置键定义
app/src/main/resources/META-INF/xposed/  # 模块声明（module.prop / java_init.list / scope.list）
.github/workflows/android.yml            # CI：push / PR 自动构建
```

## License

Apache License 2.0
