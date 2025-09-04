# 如何配置 UE5 最佳构建环境（Windows/MSVC/Rider）

目标：在保证运行时性能与较好调试体验的前提下，获得稳定、可复用、可加速的构建流水线。适用于 UE5.6+、Windows、MSVC、Rider/VS 开发环境。

核心策略（TL;DR）
- 构建配置：优先使用 Development（优化开启），配合 VS 的 C++ Dynamic Debugging（动态调试）。
- 调试开关：在需要调试的 Target 中启用 `WindowsPlatform.bDynamicDebugging = true;`。
- 产物隔离：仅对 Game/Client/Server 使用 `BuildEnvironment = TargetBuildEnvironment.Unique;`，Editor 目标保持默认（共享环境）。
- 构建加速：启用 UBA（Unreal Build Accelerator），命令行追加 `-UseUBA`。
- 缓存：启用共享 DDC（Shared Derived Data Cache），将缓存放在高速磁盘或稳定的共享盘。
- Rider 编译选择：在编辑器内调试用 Development + Editor + Win64；独立可执行调试用 Development + Game + Win64。

为什么要这么做（设计动机）
- 用 Development + 动态调试：在优化构建中也能“像 Debug 一样”可靠断点/单步/看变量，同时保留优化带来的性能。
- 仅对 Game 使用 Unique：隔离游戏与引擎中间产物，避免模型/产物冲突；而 Editor 若也设 Unique，可能影响资源发现路径（如语言包），因此让 Editor 沿用默认共享环境更稳。
- 用 UBA：显著提升首次和增量编译速度，适配本地和分布式模式。
- 用 Shared DDC：复用材质/纹理/着色器派生数据，减少导入与着色器编译的等待时间。

---

## 一、动态调试（Dynamic Debugging）

作用：在“优化构建”里提供接近 Debug 的调试体验（断点命中率高、调用栈与局部变量更可读）。

必要条件
- 工具链：Visual Studio 2022 17.14+，MSVC 工具集相应版本（启用 C++ Dynamic Debugging 预览特性）。
- 目标开关：在要调试的 Target 中设置：
  ```csharp
  WindowsPlatform.bDynamicDebugging = true;
  ```
- 构建配置：选择 Development（而不是 Debug/DebugGame）。

验证方式
- 在内联/优化较多的函数上下断点并单步：断点更稳定、调用栈更完整、Locals/Watch 变量更可见，即为生效。

经验教训
- “是否 Unique”与“是否动态调试”是两件独立的事；Unique 只影响中间产物布局，不影响动态调试。
- 动态调试需要你“正在调试的那个 Target”开启该开关并重新编译。

---

## 二、产物隔离策略（Unique 的取舍）

推荐做法
- Game/Client/Server：开启
  ```csharp
  BuildEnvironment = TargetBuildEnvironment.Unique;
  ```
  理由：彻底隔离游戏与引擎的中间产物，避免在 Rider 直接打开 `.uproject` 生成项目模型时发生冲突（尤其是在开启动态调试后，符号/PDB 更复杂）。
- Editor：保持默认（不要设置 Unique）
  理由：在部分环境下，Editor 使用 Unique 会导致资源发现路径改变，表现为“编辑器语言列表缺失/多语言丢失”。保持默认可避免该问题。

注意
- 第一次启用 Unique 会触发“冷编译”，可能需要重新编译数千个源文件，属于预期；后续增量恢复正常。
- 若曾在 Editor 上启用过 Unique，建议清理项目侧 Binaries/Intermediate 与 Saved/Localization，再构建。

---

## 三、启用 UBA（Unreal Build Accelerator）

目的：加速本地或分布式编译。

本地（单机）启用
- 命令行构建（示例）：
  ```
  Engine/Build/BatchFiles/Build.bat YourEditorTarget Win64 Development -UseUBA -WaitMutex
  ```
- RunUAT（示例）：
  ```
  Engine/Build/BatchFiles/RunUAT.bat BuildCookRun -project="YourProject.uproject" -target="YourEditorTarget" -platform=Win64 -configuration=Development -UseUBA
  ```
- Rider/VS：在 UBT 的“额外参数”里追加 `-UseUBA`。

分布式（可选）
- 结合 Horde 进行远程/分布式编译。仍然在 UBT 参数中添加 `-UseUBA`，并按 Horde 文档配置 Agent 与网络。

验证
- 构建日志出现 “Using Unreal Build Accelerator (UBA)” 或显示 Worker/Cache 信息。
- 首次会预热缓存，增量构建提速明显。

---

## 四、共享 DDC（Derived Data Cache）

目的：复用纹理、材质、着色器等派生数据，显著减少编辑器首次导入/着色器编译时间。

建议
- 启用 Shared DDC，路径选用高速 SSD 或可靠的局域网共享。
- 团队成员指向同一共享路径，可复用大部分缓存。
- 切换分支或升级引擎版本后，可先保留旧缓存，必要时清理无效项以节省空间。

---

## 五、Rider 构建/调试选择

常用组合
- 在编辑器里调试：Development + Editor + Win64
- 调试独立可执行：Development + Game + Win64
- 专用服务器：Development + Server
- 多客户端：Development + Client

说明
- 动态调试的价值体现在“优化构建”，因此首选 Development。
- 每次切换关键开关（如开启/关闭 Unique 或 Dynamic Debugging）后，建议对对应 Target 做一次“重建”。

---

## 六、切换/清理与稳定性建议

- 清理范围（项目侧）：
  - Binaries、Intermediate
  - Saved/Config/WindowsEditor 下与语言/文化相关的覆盖项（必要时备份后移除）
  - Saved/Localization（如遇到语言加载异常）
- 不动引擎侧中间产物，避免二次全量构建。
- 首次启用 Unique 或 UBA：全量耗时正常，可配合更高并行度与 UBA 缓存缓解。

---

## 七、常见问题与处置

1) 开启 Unique 后编辑器“多语言”不见了
- 现象：Editor 目标使用 Unique 时，语言列表缺失或只剩少数项。
- 原因：资源发现路径改变导致引擎本地化包（Engine/Editor locres）未被正确发现。
- 方案：让 Editor 目标保持默认（不 Unique），仅对 Game/Client/Server 使用 Unique；清理项目侧缓存后重启编辑器。

2) 项目模型生成失败或中间产物冲突
- 方案：对 Game/Client/Server 使用 Unique，清理项目侧 Binaries/Intermediate；Rider 重新生成项目模型。

3) 动态调试似乎无效
- 自检：确认正在调试的 Target 启用了 `WindowsPlatform.bDynamicDebugging = true;`，并以 Development 构建；VS 版本满足要求且已启用 C++ Dynamic Debugging；重建该 Target 以避免旧产物。

4) 首次编译非常慢
- 说明：Unique/UBA/缓存预热导致的冷编译属正常；后续增量会明显加快。建议同时启用 UBA 与 Shared DDC。

---

## 八、验证清单（上线前自查）

- [ ] 目标 Target 已开启 `WindowsPlatform.bDynamicDebugging = true;`
- [ ] Rider 选择 Development +（Editor 或 Game）+ Win64，并已重建该 Target
- [ ] Game/Client/Server 使用 Unique；Editor 不使用 Unique
- [ ] 构建参数包含 `-UseUBA`，日志出现 UBA 信息
- [ ] 启用了 Shared DDC，路径可写且稳定
- [ ] 语言列表完整可见（若异常，已清理项目侧缓存并恢复 Editor 的共享环境）
- [ ] 冷编译完成后，增量编译/调试体验达到预期

---

## 九、常用命令与参数速查

- 构建（编辑器）：
  ```
  Engine/Build/BatchFiles/Build.bat YourEditorTarget Win64 Development -UseUBA -WaitMutex
  ```
- 构建并运行（示例）：
  ```
  Engine/Build/BatchFiles/RunUAT.bat BuildCookRun -project="YourProject.uproject" -target="YourEditorTarget" -platform=Win64 -configuration=Development -UseUBA
  ```
- 强制文化（调试用）：
  ```
  -culture=en
  ```
  或
  ```
  -language=zh-Hans
  ```

---

结语
- 这套配置（Development + 动态调试 + UBA + 产物隔离仅限 Game + 共享 DDC）在实际项目中能兼顾性能、调试体验与稳定性。遇到问题优先回到“Editor 不 Unique、清理项目侧缓存、重建目标”的基线，再逐项验证。
