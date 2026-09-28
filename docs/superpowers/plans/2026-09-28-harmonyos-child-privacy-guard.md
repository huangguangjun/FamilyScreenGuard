# HarmonyOS 儿童隐私守护 App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个在纯血鸿蒙手机本地运行、在时段内检测登记儿童并以 PIN 保护隐私遮罩的 App。

**Architecture:** ArkTS/ArkUI 单模块 Stage 应用。`GuardianSession` 是唯一守护状态机，协调调度、相机和遮罩。相机帧只在内存内流向可替换的离线 `FaceMatcher`；重启恢复只重载规则和下一时段，绝不静默开启相机。

**Tech Stack:** HarmonyOS NEXT、ArkTS、ArkUI、Ability Kit、Notification Kit、Camera Kit、系统安全存储、DevEco Studio 测试。

**Spec:** `docs/superpowers/specs/2026-09-28-harmonyos-child-privacy-guard-design.md`

## Global Constraints

- 普通第三方 HarmonyOS NEXT App；不得申请 MDM、系统签名或后台不可见启动权限。
- 所有照片、人脸特征与事件记录只保存在本机；不得上传、录制或保存实时帧。
- PIN 只保存加盐慢哈希；照片与特征使用系统安全存储。
- 相机只能在可见守护会话内启动；相机、权限或模型异常必须显示“保护未生效”。
- 重启后只恢复规则；若系统没有可用的开机回调，首次打开 App 时恢复并明确提示监护人。

---

## File structure

| 路径 | 职责 |
| --- | --- |
| `AppScope/app.json5` | 应用级配置 |
| `entry/src/main/module.json5` | Ability、运行时权限和通知配置 |
| `entry/src/main/ets/domain/*.ets` | 类型、时段计算和状态机 |
| `entry/src/main/ets/data/*.ets` | PIN、规则、儿童资料和事件的安全存储 |
| `entry/src/main/ets/services/*.ets` | 调度、通知、恢复、相机、人脸比对、守护会话 |
| `entry/src/main/ets/pages/*.ets` | 设置、首页、守护与拦截页面 |
| `entry/src/test/*.test.ets` | 领域和服务测试 |
| `docs/verification/*.md` | 真机能力探针和发布验收 |

### Task 1: 创建工程并验证目标设备能力

**Files:** Create `AppScope/app.json5`, `entry/src/main/module.json5`, `entry/src/main/ets/entryability/EntryAbility.ets`, `docs/verification/target-device-capability-probe.md`.

**Produces:** 可安装的最小 Stage 应用和能力实测表。

- [ ] 在 DevEco Studio 创建 Empty Ability 工程，包名为 `com.huangguangjun.familyscreenguard`；只声明 `ohos.permission.CAMERA` 与通知所需运行时权限。
- [ ] 创建探针表：前置低分辨率预览、通知、重启后规则保留、后台限制。每项记录设备型号、系统/API 版本和通过/失败。
- [ ] 在目标机重启后测试：允许的结果是“规则仍在，首次启动 App 时重算下一时段”；不把静默自启相机标为通过条件。
- [ ] 提交：`git add AppScope entry docs/verification; git commit -m "chore: scaffold app and capability probe"`。

### Task 2: 实现时段计算与守护状态机

**Files:** Create `entry/src/main/ets/domain/Models.ets`, `ScheduleCalculator.ets`, `GuardianStateMachine.ets`; Test `entry/src/test/ScheduleCalculator.test.ets`, `GuardianStateMachine.test.ets`.

**Produces:** `ScheduleCalculator.nextOccurrence(now, rules)` 和 `GuardianStateMachine.transition(event)`。

- [ ] 写失败测试：周一 23:00–01:00 的规则在周一 23:30 为激活状态；`INTERCEPTED` 收到 `UNLOCK_FAILED` 后仍是 `INTERCEPTED`。
- [ ] 实现如下领域契约与转换表：

```ts
export enum GuardState { IDLE, COUNTDOWN, GUARDING, INTERCEPTED, PAUSED, UNAVAILABLE }
export interface GuardRule { weekdays: number[]; startMinute: number; endMinute: number }
export type GuardEvent = { type: 'TIME_STARTED' } | { type: 'FACE_CONFIRMED' } |
  { type: 'UNLOCK_OK' } | { type: 'UNLOCK_FAILED' } | { type: 'TIME_ENDED' }
```

- [ ] 运行测试，覆盖工作日、跨午夜、时段结束和未知事件；全部通过后提交：`git add entry/src/main/ets/domain entry/src/test; git commit -m "feat: add schedule and guard state machine"`。

### Task 3: 实现加密本地存储与 PIN

**Files:** Create `entry/src/main/ets/data/SecureStore.ets`, `PinService.ets`, `AuditLog.ets`; Test `entry/src/test/PinService.test.ets`.

**Produces:** `PinService.create(pin)`, `PinService.verify(pin)`, `SecureStore.saveRules(rules)`。

- [ ] 写失败测试，验证建立 `318642` 后，`318643` 返回 false、正确 PIN 返回 true。
- [ ] 以随机 salt 加慢哈希保存 PIN；安全存储只接收规则、照片引用、派生特征和 `{ occurredAt, kind, detailCode }` 事件。禁止事件类型出现 `frame`、`image` 或 `embedding` 字段。
- [ ] 运行测试并手动检查日志内容不含照片或 PIN；提交：`git add entry/src/main/ets/data entry/src/test; git commit -m "feat: add secure local settings and PIN"`。

### Task 4: 添加儿童档案与本地照片选择

**Files:** Create `entry/src/main/ets/services/ChildProfileService.ets`, `entry/src/main/ets/pages/ChildProfilesPage.ets`; Test `entry/src/test/ChildProfileService.test.ets`.

**Produces:** `addPhoto(childId, uri)`, `enableProfile(childId)`, `deleteChild(childId)`。

- [ ] 写失败测试：少于 3 张参考照片时 `enableProfile` 抛出 `MINIMUM_REFERENCE_PHOTOS`。
- [ ] 使用系统照片选择器，而非整库媒体读取；启用时限制为 3–5 张。删除儿童必须在同一操作中删除照片引用、特征和其事件摘要。
- [ ] 验证导入、删除和 3–5 张限制后提交：`git add entry/src/main/ets/services/ChildProfileService.ets entry/src/main/ets/pages/ChildProfilesPage.ets entry/src/test; git commit -m "feat: manage child reference profiles"`。

### Task 5: 建立离线人脸匹配边界并完成模型验证

**Files:** Create `entry/src/main/ets/services/FaceMatcher.ets`, `NoopFaceMatcher.ets`, `ConsecutiveMatchGate.ets`; Test `entry/src/test/ConsecutiveMatchGate.test.ets`; Modify `docs/verification/target-device-capability-probe.md`.

**Produces:** `FaceMatcher.enroll(photoUri)`, `FaceMatcher.match(frame)`，以及连续命中判定。

- [ ] 写失败测试：阈值 `0.86`、连续次数 `3` 时，前两次 `0.90`/`0.91` 不触发，第三次 `0.92` 触发。
- [ ] 定义接口：

```ts
export interface MatchResult { status: 'MATCH' | 'NO_MATCH' | 'UNAVAILABLE'; childId?: string; score?: number }
export interface FaceMatcher { enroll(photoUri: string): Promise<string>; match(frame: ArrayBuffer): Promise<MatchResult> }
```

- [ ] `NoopFaceMatcher` 必须返回 `UNAVAILABLE`。仅在官方设备端 API 或许可证明确的离线模型已完成真机测试后，才加入真实实现；记录模型版本、许可证、离线证明、耗时和弱光结果。没有合规实现则停止在“保护未生效”，不得伪造识别功能。
- [ ] 测试门控并提交：`git add entry/src/main/ets/services entry/src/test docs/verification; git commit -m "feat: add offline face matching boundary"`。

### Task 6: 接入前置相机低频采样

**Files:** Create `entry/src/main/ets/services/CameraSampler.ets`, `CameraSessionController.ets`; Test `entry/src/test/CameraSampler.test.ets`.

**Produces:** `startFrontCamera(onFrame)`, `stop()` 和 `CameraAvailability`。

- [ ] 写失败测试：3 秒采样间隔时，0 ms、1000 ms、3000 ms 三帧中仅第 1、3 帧传给回调。
- [ ] 使用 Camera Kit 前置低分辨率预览流；默认每 3 秒取一帧、传给回调后释放。启动失败、权限撤回或被占用时返回 `UNAVAILABLE`，不写入文件或日志。
- [ ] 真机运行 30 分钟，记录相机指示、内存、发热及停止后资源释放；通过测试后提交：`git add entry/src/main/ets/services entry/src/test docs/verification; git commit -m "feat: sample front camera at low frequency"`。

### Task 7: 实现通知、倒计时、延后与重启恢复

**Files:** Create `entry/src/main/ets/services/ScheduleService.ets`, `NotificationService.ets`, `BootRecovery.ets`; Modify `entry/src/main/ets/entryability/EntryAbility.ets`; Test `entry/src/test/ScheduleService.test.ets`.

**Produces:** `arm()`, `delayOnce(15)`, `recoverAfterBoot()`。

- [ ] 写失败测试：对同一时段调用两次 `delayOnce(15)`，第二次抛出 `DELAY_ALREADY_USED`。
- [ ] 到点发出带 60 秒倒计时的通知；经 PIN 验证后允许立即开始或仅一次延后 15 分钟；超时进入守护。`BootRecovery` 只读取规则并调用 `arm()`，不得调用相机或强行打开 UI。
- [ ] 真实设备上测试重启、首次启动恢复、通知操作和超时自动开始，记录实际系统触发点；提交：`git add entry/src/main/ets/services entry/src/main/ets/entryability entry/src/test docs/verification; git commit -m "feat: schedule sessions and restore rules"`。

### Task 8: 组合为守护会话

**Files:** Create `entry/src/main/ets/services/GuardianSession.ets`; Test `entry/src/test/GuardianSession.test.ets`.

**Produces:** `start()`, `pause(pin)`, `unlock(pin)`, `end()` 和可观察的 `GuardState`。

- [ ] 写失败测试：相机启动失败后状态为 `UNAVAILABLE`，而不是 `GUARDING`。
- [ ] 仅在 `GUARDING` 启动相机；连续匹配进入 `INTERCEPTED`；相机或模型错误变为 `UNAVAILABLE` 并写无图像事件；结束、暂停和时段结束均调用 `camera.stop()`。
- [ ] 运行完整状态转换测试并提交：`git add entry/src/main/ets/services/GuardianSession.ets entry/src/test; git commit -m "feat: coordinate guardian session"`。

### Task 9: 构建 ArkUI 页面与受 PIN 保护的交互

**Files:** Create `entry/src/main/ets/pages/SetupPage.ets`, `HomePage.ets`, `GuardPage.ets`, `InterceptPage.ets`, `entry/src/main/ets/components/PinSheet.ets`; Test `entry/src/test/GuardianUiFlow.test.ets`.

**Produces:** 设置、首页、守护页和黑色拦截页。

- [ ] 写失败 UI 测试：拦截页输入错误 PIN 后依然可见。
- [ ] 初始设置收集 PIN、权限、照片和时间规则；守护页不展示相机预览；拦截页为黑色全屏，显示“请不要偷看他人的手机”。暂停、设置和解除一律走 `PinSheet`。
- [ ] 检查中文文案、字体缩放、屏幕阅读器标签、返回手势和“保护未生效”页；提交：`git add entry/src/main/ets/pages entry/src/main/ets/components entry/src/test; git commit -m "feat: add PIN-protected guard interface"`。

### Task 10: 隐私审查、端到端验收和发布

**Files:** Create `README.md`, `docs/privacy.md`, `docs/verification/release-checklist.md`.

**Produces:** 发布说明、隐私声明和已填写的真机验收单。

- [ ] 验收单逐项覆盖：权限、3–5 张照片、跨午夜、1 分钟超时、一次延后、连续命中、错误 PIN、相机占用、30 分钟运行、重启规则恢复、时段结束释放、删除全部数据。
- [ ] 跑全部自动化测试并在目标机按清单回归；未通过项必须修复或从用户承诺中删除。
- [ ] 检查网络调用、日志与构建目录，确认无实时帧、特征、明文 PIN 或签名密钥；隐私声明明确“不替代系统锁屏”和“不静默后台开启摄像头”。
- [ ] 构建签名 `.hap`，将设备型号、系统版本和构建结果写入验收单；提交：`git add README.md docs; git commit -m "docs: add privacy and release verification"`。

## Plan self-review

- **规格覆盖：** 任务 2、7 覆盖时间规则、倒计时、延后、跨午夜和重启恢复；任务 3、4 覆盖 PIN、照片、加密与删除；任务 5、6、8 覆盖本地匹配、低频相机、连续命中和故障降级；任务 9 覆盖黑色遮罩；任务 1、10 覆盖真机与发布验证。
- **无虚假承诺：** 开机恢复与离线识别均有明确的设备验证和停止条件，不把后台相机或未知 API 写成既定能力。
- **接口一致性：** UI 只通过 `GuardianSession` 改变守护状态；`GuardianSession` 只依赖任务 2、5、6 定义的状态机、匹配器和相机控制器。
