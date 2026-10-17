# 《末日十州》缺陷修复交付 · Claude → Codex

日期：2026-10-16 · 执行者：Claude（缺陷修复）· 接收方：Codex（独立回归与整合）

## 0. 版本边界（先读）

**本交付不是QA02302修复。** 协作资料页只提供了QA02301历史源码。QA02302的确切源码、Web包、补丁、原始日志、原工作区HANDOFF和最新七份管理文档都没有提供（页面`manifest.json`中`"available_code": "QA02301 only"`）。

因此：

- 下面所有复现、补丁和测试都基于 **QA02301 Review 源码**，标为“QA02301草案补丁，待在QA02302上重放验证”；
- 补丁**没有**合入任何交付基线，也不能作为最新版修复；
- 我没有修改QA02302本轮改过的两个文件（`systems/FamilyQuest.gd`、`ui/QuestMenus.gd`），也没有审查其中的QA02302-01修复。这两个文件归Codex。

| 项目 | 值 |
| --- | --- |
| 实际使用的基线 | `LastDawn-0.2.3-QA02301-ReviewSource.zip`，SHA-256 `6106610ec0297176f28146afc52bc88dee15ac2e8776134df6ff92f3e853908e`（与页面`SHA256SUMS.txt`一致） |
| QA02302报告所称基线 | `LastDawn-0.2.3-QA02301-Source.zip`，SHA-256 `8fc023e9…0c7f`（另一份打包，我没有拿到，无法逐文件比对） |
| 引擎 | Godot `4.6.3.stable.official.7d41c59c4`（官方Linux包，SHA-512已按官方`SHA512-SUMS.txt`核对；与报告所用版本相同） |
| 运行环境 | 云端Linux容器，headless + `gl_compatibility`。没有浏览器导出，没有手机 |

在QA02302上重放前，先核对这三个文件。若SHA-256与下表一致（报告称QA02302未改动它们），补丁可原样应用：

| 文件 | QA02301 SHA-256 |
| --- | --- |
| `scripts/WorldManager.gd` | `ab6ecf5089851bc83130394a65f719ca937e5683d2d6e0014a48a175b0280ab8` |
| `ui/SurvivalHUD.gd` | `7bb0326213819ad4ef5e7c13df9f9ef77066a7b12722fa694d8d2a931df51d18` |
| `systems/SaveManager.gd` | `a48cbb430a5797a6d547493fc9368f1f7d07705d8d944a8df59ccd1309429841` |

**环境可信度：** 未改动的QA02301基线在本环境全套跑通，806项引擎检查 + 9项静态检查，各套件exit 0，与历史806/9一致（`logs/baseline-qa02301/`）。首次导入出现的4条字体缓存错误、二次导入恢复正常，也同样复现，与报告的P3工具链问题一致。

## 1. 问题清单

| 编号 | 严重度 | 类别 | 状态 | 本轮处理 |
| --- | --- | --- | --- | --- |
| LD-01 | P1 | 丢档 | 引擎内已复现 | 已修复（草案补丁）+ 回归测试 |
| LD-02 | P2 | 丢档风险 | 引擎内已复现 | 需产品决定，未改 |
| LD-03 | P3 | 脚本错误 | 已复现 | 已修复（草案补丁）+ 回归测试 |
| LD-04 | 疑点 | Web存档落盘 | 只读引擎源码，未运行 | 需iPhone真机验证，未改 |
| LD-05 | 疑点 | 流程阻断 | 只读代码，未复现 | 范围决定，未改 |

### LD-01（P1，丢档）标题页切到后台会覆盖真实存档

- **文件/函数：** `scripts/WorldManager.gd`：`_web_visibility_changed`、`_notification(NOTIFICATION_APPLICATION_PAUSED)`、`_process`中的自动保存。
- **触发条件：** 本机已有存档。打开游戏后停在欢迎页（还没点“继续本机存档”），这时页面切到后台（Safari切App、锁屏、切标签页）或Android应用暂停。
- **原因：** 欢迎页下运行的是全新的默认世界；`ready_for_saves`在`_ready`末尾就已为true；Web的`visibilitychange`回调不受树暂停影响。于是`SaveManager.save_world`把空白世界写入主档，并把真实进度轮换到`.bak`。
- **影响：** 切后台一次，主档变成空白进度，此时点“继续本机存档”读到的是空档；再过一次自动保存或再切一次后台，`.bak`也被空档覆盖，真实进度彻底丢失。手机上看到标题后先切出去、回来再继续，是很常见的操作。
- **复现（引擎内，已执行）：** `tests/SessionGuardTests.gd`。先存两次真实进度（木材17），模拟页面重载（重置autoload），打开标题，触发两次`NOTIFICATION_APPLICATION_PAUSED`。在QA02301上，主档和备份的木材都变为0，“继续”读出0（`logs/sessionguard-base/SessionGuardTests.log`，5项FAIL，exit 1）。
  - Web的`visibilitychange`路径与暂停路径调用同一个`save_world`，代码相同；但headless环境没有JavaScriptBridge，**这条路径没有在浏览器中实际运行**。
- **修复：**
  - `WorldManager`新增`awaiting_session_choice`。标题页由`open_title()`打开时置为true；`can_autosave()`在此期间拒绝自动、后台和暂停三种写入。
  - 成功读档（`restore_snapshot`末尾）或关闭任何模态窗口（`SurvivalHUD.close_modal`）后恢复写入。“进入灰松郊区”、读档、经设置页返回都会正常解除，不会永久阻塞保存。
  - headless测试不打开标题，该标志保持false，原有测试行为不变。
- **补丁：** `patches/LD-01-title-screen-save-guard.patch`（WorldManager.gd +13/−4，SurvivalHUD.gd +1）。

### LD-02（P2，丢档风险，未修，需要产品决定）在欢迎页点“进入灰松郊区”会静默覆盖已有存档

- **触发条件：** 已有存档时，点了欢迎页上的强调按钮“进入灰松郊区”，而不是“继续本机存档”。
- **实测（`logs/ld02-probe/SessionGuardTests.log`）：** 第1次自动保存（45秒）后，主档为0，备份仍为17。但“读取存档”优先读有效的主档，所以UI已无法找回进度。第2次自动保存（90秒）后，主档和备份都为0。
- **为何未修：** “进入”是否就是“新游戏”，属于流程和文案决定。全程没有任何覆盖提示。
- **可选方案（任选其一，均不新增玩法）：**
  - 已有存档时，“进入灰松郊区”先弹一次确认：“将开始新的进度，并在45秒后覆盖本机存档”；
  - 已有存档时，把“继续本机存档”设为强调或首位按钮。
- 请项目管理方决定后，由我实施。

### LD-03（P3，脚本错误）旧版存档的`state_id`为非字符串时触发SCRIPT ERROR

- **文件/函数：** `systems/SaveManager.gd`：`valid_snapshot`，最后一行`data.get("state_id") == "state_01"`。
- **触发：** schema 1/2存档中`state_id`是数字（例如`5`）。schema 3在`_complete_current`中已先检查类型，不受影响。
- **影响：** 引擎报出`SCRIPT ERROR: Invalid operands 'int' and 'String' in operator '=='`，函数异常中止，结果仍是拒绝，所以不会导致丢档。但测试运行器会把它判为失败，日志里也多出一条脚本错误。
- **修复：** 先判断`state_id is String`再比较。补丁为`patches/LD-03-legacy-state-id-type.patch`（+2/−1）。

### LD-04（疑点，需真机）Web端切后台时写入的存档可能来不及落到IndexedDB

- **依据（只读源码，未运行）：** Godot 4.6.3 `platform/web/os_web.cpp`的`main_loop_iterate()`只有在主循环迭代时，才把`idb_needs_sync`推送到IndexedDB；主循环由requestAnimationFrame驱动，而页面隐藏后浏览器会暂停rAF。`JavaScriptBridge.force_fs_sync()`也只是置位这个标志。
- **影响推测：** `visibilitychange(hidden)`时写入的存档，要等页面重新可见后才同步。如果iOS在后台直接回收了页面，这次存档可能丢失；最坏回退到上一次45秒自动保存的进度。
- **验证方法：** 在iPhone Safari上，移动并拾取物品后立即切到其他App；再用另一个App占满内存迫使Safari回收页面；回到游戏后点“继续”，核对物品数量。
- 本轮未改：游戏侧没有可靠的同步刷盘方式，修复需要引擎层面的改动，不属于最小修复。

### LD-05（疑点，P3）玩家建墙围住小雨后，撤离任务可能永久阻断

- `BuildingSystem.check_place`只阻止与角色本体重叠，并不阻止围住角色；游戏也没有拆除功能。
- 需要玩家刻意围堵才会触发，未复现。加入拆除属于新增玩法，本轮不处理，仅登记。

### 未覆盖：QA02302-01（失能门禁）

QA02302源码不在手，`FamilyQuest.gd`和`QuestMenus.gd`归Codex。我既没有审查也没有修改这两个文件。

## 2. 修改文件与归属

| 文件 | 改动 | 是否与Codex冲突 |
| --- | --- | --- |
| `scripts/WorldManager.gd` | LD-01 | 报告称QA02302未改动；请先按上表核对SHA |
| `ui/SurvivalHUD.gd` | LD-01（1行） | 同上 |
| `systems/SaveManager.gd` | LD-03（2行） | 同上（报告称存档实现未改） |
| `tests/SessionGuardTests.gd/.tscn` | 新增文件 | 无 |
| `tests/run_phase2_complete.sh` | **未改**（QA02302改过测试脚本，避免冲突） | 请Codex在整合时把`SessionGuardTests`加入套件循环 |

我没有改动：`systems/FamilyQuest.gd`、`ui/QuestMenus.gd`、任何测试运行脚本、导出配置、管理文档、原源码ZIP，也没有改动稳定版或用户存档。

## 3. 测试命令与结果（全部实际执行）

环境变量统一为：`PATH`包含Godot 4.6.3（命令名`godot`），并通过`QA_LOG_ROOT`和`QA_PROFILE_ROOT`隔离日志与用户目录。

| # | 对象 | 命令 | 退出码 | 结果 | 日志 |
| --- | --- | --- | --- | --- | --- |
| 1 | QA02301原版 | `bash tests/run_phase2_complete.sh` | 0 | 806引擎 / 0失败；9静态 / 0失败 | `logs/baseline-qa02301/` |
| 2 | QA02301原版 + 新测试 | `python3 tests/run_qa_suite.py SessionGuardTests` | **1** | 13项中5项FAIL + 1条SCRIPT ERROR（修复前） | `logs/sessionguard-base/` |
| 3 | 打补丁后 | 同上 | 0 | 13 / 0失败（修复后） | `logs/sessionguard-fix/` |
| 4 | 从原ZIP重新解压，`patch -p1`应用3个补丁 | `bash tests/run_phase2_complete.sh` 后接 `run_qa_suite.py SessionGuardTests` | 0 / 0 | 819引擎（806 + 13）/ 0失败；9静态 / 0失败 | `logs/patched-from-zip/` |
| 5 | LD-02探测 | 一次性改写的SessionGuard | — | 第1次自动保存后：主档0、备份17；第2次后：0 / 0 | `logs/ld02-probe/` |

说明：

- 第4组`suite-status.json`的`import`条目中有4条字体缓存ERROR，是已知的首次导入问题，`import-second`为exit 0且无错误，与报告P3一致，与本补丁无关。
- 三个补丁都通过了`patch --dry-run`；应用后文件与工作副本逐字节一致。

## 4. 剩余限制

1. **全部结论限于QA02301。** 拿到QA02302源码后，需要先核对SHA，再重放补丁并重跑全套。
2. 没有导出Web，没有运行浏览器自动化，没有iPhone或Android真机。Web的`visibilitychange`路径只靠读代码确认与已测路径等价。
3. LD-02需要产品决定；LD-04需要真机验证；LD-05属于范围决定。
4. 我没有重跑QA02302报告中的814/64项结果，不对这些数字背书。
5. 未构建APK，未操作GitHub，未部署。

## 5. 回退

- 在工程根目录（`LastDawn`的上一级）执行`patch -R -p1 < patches/<名称>.patch`，三个补丁各自独立回退；
- 再删除`tests/SessionGuardTests.gd`、`tests/SessionGuardTests.tscn`，以及Godot生成的`.uid`文件；
- 本补丁不改变存档格式（仍为schema 3，字段不变），回退不影响已有存档。

## 6. 交付物

```
claude-qa-handoff/
  REPORT.md                                        本文件
  patches/LD-01-title-screen-save-guard.patch      WorldManager.gd + SurvivalHUD.gd
  patches/LD-03-legacy-state-id-type.patch         SaveManager.gd
  patches/LD-01-03-regression-SessionGuardTests.patch  新增测试（也以原文件放在tests/）
  tests/SessionGuardTests.gd / .tscn
  logs/…                                           原始日志与suite-status.json（含退出码）
  SHA256SUMS
```

应用方法：在包含`LastDawn/`的目录执行`patch -p1 < claude-qa-handoff/patches/<名称>.patch`。
