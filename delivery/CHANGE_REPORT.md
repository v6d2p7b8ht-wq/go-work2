# 变更报告 · 末日十州 QA02302 重建版（Claude）

日期：2026-10-17 · 状态：第二阶段测试候选。**第二阶段未完成**：iPhone Safari与Android真机验收都待进行。

## 1. 版本标识

| 项目 | 值 |
| --- | --- |
| 版本 | **QA02302 重建版（Claude）**，不是找回的QA02302原件；内部`config/version="0.2.3-test"` |
| 起点 | `LastDawn-0.2.3-QA02301-ReviewSource.zip`，SHA-256 `6106610ec0297176f28146afc52bc88dee15ac2e8776134df6ff92f3e853908e` |
| 交付源码 | `LastDawn-0.2.3-QA02302-RECONSTRUCTED-Claude-Source.zip`，441个文件，SHA-256见`SHA256SUMS.txt` |
| 测试所用源码 | SHA-256 `1fd6197c43b23a91c7b752e9a00f83e055412668ea20bf1bd6e51e9452eb64ca`。与交付ZIP只差`HANDOFF.md`（文档补充），其余440个文件逐字节相同；见`evidence/SOURCE-MANIFEST-*.sha256` |
| 引擎 | Godot `4.6.3.stable.official.7d41c59c4` |
| Web ZIP | **未构建**（没有导出模板，未导出） |
| APK | 未构建 |

## 2. 各项状态

| 项目 | 状态 | 证据（本轮实际执行） |
| --- | --- | --- |
| 标题页及切后台存档保护（LD-01） | **已修复，引擎内验证**：标题页期间自动、后台、手动保存都不写盘；主档和备份逐字节不变 | SessionGuardTests |
| 关闭设置或其他菜单不会意外开启保存 | **已修复，引擎内验证**：关闭设置、确认页或标题页的×都回到标题页，保护保持 | SessionGuardTests |
| 已有进度时开始新游戏需确认，取消后两个存档不变（LD-02） | **已修复，引擎内验证**：确认页；取消或关闭确认页后两个文件逐字节不变；只有确认后才替换 | SessionGuardTests |
| 旧档`state_id`类型校验（LD-03） | **已修复，引擎内验证** | SessionGuardTests末项 |
| 失能时家人等待/领奖（QA02302-01重建） | **已修复，引擎内验证**：命令被拒绝，按钮禁用；恢复后奖励只发一次 | FamilyIncapacitationGateTests |
| 切后台时的触控清理 | 保存被阻止时仍会执行 | SessionGuardTests |

以上都是Godot headless引擎测试的结果，没有在浏览器或手机上验证。

## 3. 未解决

1. iPhone Safari与Android真机验收，以及Web导出和浏览器自动化，都未进行。
2. 浏览器脚本中，在已有存档的档案上点击“进入灰松郊区”的流程，需要改为“开始新游戏”→“确认开始新游戏”。脚本未修改，也未运行。
3. LD-04（疑点）：Web切后台时写入的存档，要等页面重新可见才会同步到IndexedDB。这是根据引擎源码判断的，需要iPhone验证，未修改。
4. LD-05（疑点）：玩家可以建墙围住小雨，而游戏没有拆除功能。属于范围决定，未修改。
5. 首次导入会出现4条字体缓存ERROR，二次导入正常（P3工具链问题，已有记录）。
6. QA02302原有的`PhaseTwoFlowTests`和浏览器脚本修正无法从报告还原；本包用自己的测试代替，检查项不一一对应。
7. 标题页的×现在不做任何事（标题页不可关闭），需要真机确认体验。

## 4. 最小补丁（`evidence/patches/`）

- `01-runtime-LD01-LD02-LD03-QA02302-01.patch`：5个运行时文件，+67/−18。
- `02-tests-SessionGuard-FamilyGate.patch`：2个新套件，并加入`run_phase2_complete.sh`。
- `03-docs-handoff-management.patch`：HANDOFF、README和七份管理文档。
- `incremental-from-draft-v1-runtime.patch`：相对上一版草案的运行时增量。

在QA02301起点的全新解压目录中，在`LastDawn/`的上一级执行`git apply -p1`应用01→03，结果与交付ZIP的解压目录`diff -r`完全一致（本轮已实际核对）。

## 5. 测试证据

### 5.1 本轮实际执行（Claude，云端Linux，Godot 4.6.3 headless）

原始日志都在`evidence/logs/`，各目录中的`suite-status.json`记录退出码。失败日志全部保留。

| # | 被测对象 | 命令 | 退出码 | 结果 | 日志 |
| --- | --- | --- | --- | --- | --- |
| A | QA02301原版（对照） | `bash tests/run_phase2_complete.sh` | 0 | 806引擎 / 0失败；9静态 / 0失败 | `baseline-qa02301/` |
| B | QA02301原版 + 新套件 | `python3 tests/run_qa_suite.py SessionGuardTests` | 1 | 24项中18项失败，另有1条SCRIPT ERROR | `cmp-orig/` |
| C | 同上 | `python3 tests/run_qa_suite.py FamilyIncapacitationGateTests` | 1 | 11项中5项失败 | `cmp-orig/` |
| D | 上一版草案 + 新套件 | 同B | 1 | 24项中16项失败（设置关闭绕过、阻止保存时不清理触控等） | `cmp-draft1/` |
| E | 上一版草案 + 新套件 | 同C | 1 | 11项中5项失败 | `cmp-draft1/` |
| F | 测试所用ZIP（`1fd6197c…`）全新解压、全新导入 | `bash tests/run_phase2_complete.sh` | **0** | **841引擎 / 0失败**；**9静态 / 0失败**；首次导入4条已知ERROR，二次导入干净 | `final-from-zip/` |
| G | 运行时字形扫描 | `godot --headless --path . --script tools/runtime_glyph_scan.gd` | 0 | 无缺失字形 | `glyph-scan-final.log` |

开发过程中保留的失败记录：有一次全套运行因确认页文案使用了内嵌字体缺少的字，`PerformanceTests`字形检查失败1项，`run_phase2_complete.sh`以exit 1结束。那次的日志目录后来被重跑覆盖，没有保留原始文件，只有本段文字记录。文案改写后，F组通过。

### 5.2 历史结果（不是本轮执行，仅供对照，未独立复测）

- QA02301交付记录：806引擎、9静态、59项浏览器检查通过（交付方日志）。
- QA02302报告：814引擎、9静态、64项桌面Web检查通过，PCK `b838c862…a041`。这些对应遗失的原件，与本重建版无关。

## 6. 交付文件

| 附件 | 内容 |
| --- | --- |
| `LastDawn-0.2.3-QA02302-RECONSTRUCTED-Claude-Source.zip` | 完整可运行的Godot源码：project.godot、全部场景/资源/脚本、HANDOFF.md、七份管理文档 |
| `LastDawn-QA02302R-Claude-Evidence.zip` | 补丁、原始日志、新测试副本、字形扫描工具、逐文件清单、本报告 |
| `CHANGE_REPORT.md` | 本报告 |
| `SHA256SUMS.txt` | 上述每个交付文件的SHA-256 |

回退：在源码目录执行`patch -R -p1 < docs/QA02302R-runtime.patch`，或直接改用QA02301起点ZIP。存档格式没有变化（schema 3）。下一任务见`HANDOFF.md`第8节。
