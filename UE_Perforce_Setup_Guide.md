# 虚幻引擎（Unreal Engine）集成 Perforce Helix Core 版本控制配置指南

本文档旨在为使用虚幻引擎（Unreal Engine）的开发者提供一个清晰、完整的 Perforce Helix Core 版本控制系统配置流程。Perforce 是 Epic Games 官方推荐的版本控制工具，尤其擅长处理大型二进制文件（如 `.uasset` 和 `.umap`），是团队协作开发的理想选择。

---

### 核心步骤概览

1.  **安装 Perforce 服务器与客户端**：获取并安装 Helix Core 服务器 (P4D) 和可视化客户端 (P4V)。
2.  **配置 Perforce 服务器**：创建仓库 (Depot)、用户，并为 UE 项目设置特定的文件类型映射 (Typemap)。
3.  **创建并配置工作区 (Workspace)**：在本地计算机上设置一个目录，用于同步服务器文件。
4.  **配置忽略文件 (.p4ignore)**：告诉 Perforce 哪些 UE 生成的临时文件不需要纳入版本控制。
5.  **上传项目到 Perforce**：将您的 UE 项目首次提交到服务器。
6.  **在虚幻引擎中连接**：在 UE 编辑器内部连接到 Perforce 服务器。

---

### 详细操作步骤

#### 第1步：安装 Perforce 服务器与客户端 (P4D & P4V)

1.  **下载软件**：
    *   访问 [Perforce 官网](https://www.perforce.com/downloads/helix-core) 下载 **Helix Core Server (P4D)** 和 **Helix Visual Client (P4V)**。
    *   对于小型团队（最多5个用户），Perforce 提供免费许可。

2.  **安装服务器 (P4D)**：
    *   运行 P4D 安装程序，建议接受默认配置。
    *   记下服务器地址（本机通常是 `localhost:1666`）和创建的超级用户信息。
    *   **重要建议**：为避免潜在问题，Epic Games 推荐将 Perforce 服务器设置为**不区分大小写 (case-insensitive)**。这通常在服务器首次初始化时进行配置。

3.  **安装客户端 (P4V)**：
    *   在您的开发工作站上运行 P4V 安装程序，按提示完成安装即可。

#### 第2步：配置 Perforce 服务器

1.  **连接服务器**：
    *   打开 P4V 客户端。
    *   使用您的服务器地址（例如 `localhost:1666`）和管理员账户登录。

2.  **创建仓库 (Depot)**：
    *   在 P4V 顶部菜单，选择 `View -> Depots`。
    *   在 `Depots` 窗口的空白处右键，选择 `New Depot...`。
    *   为您的项目仓库命名（例如 `MyUEProject`），类型建议选择 `stream`（推荐）或 `local`。

3.  **配置类型映射 (Typemap)**：
    *   **这是最关键的一步**，它能确保 UE 的二进制文件在提交时被正确地锁定，避免多人同时修改导致冲突。
    *   在 P4V 菜单栏选择 `Tools -> Open Command Window`。
    *   在弹出的命令行窗口中输入 `p4 typemap` 并回车。
    *   系统会用默认文本编辑器打开类型映射文件。将以下内容**完整地粘贴到文件末尾**，然后保存并关闭文件。

    ```
    # Perforce File Type Mapping for Unreal Engine
    TypeMap:
        binary+w //....uasset
        binary+w //....umap
        binary+w //....upk
        binary+w //....udk
        binary+w //....ubulk
        binary+w //....ufont
        binary+w //....ushaderbytecode
        binary+w //....png
        binary+w //....tga
        binary+w //....bmp
        binary+w //....ico
        binary+w //....psd
        binary+w //....wav
        binary+w //....ogg
        binary+w //....mp3
        binary+w //....fbx
        binary+w //....obj
        binary+w //....ma
        binary+w //....mb
        binary+w //....max
        binary+w //....3ds
        binary+w //....mov
        binary+w //....wmv
        binary+w //....avi
        binary+w //....mp4
        text //....ini
        text //....config
        text //....cpp
        text //....h
        text //....c
        text //....cs
        text //....m
        text //....mm
        text //....py
        text //....pl
        text //....pm
        text //....js
        text //....css
        text //....htm
        text //....html
        text //....xml
        text //....xsl
        text //....txt
        text //....log
        text //....md
    ```

#### 第3步：创建并配置工作区 (Workspace)

工作区 (Workspace) 是您本地硬盘上的一个目录，它与服务器上的仓库 (Depot) 建立映射关系。

1.  在 P4V 菜单栏选择 `View -> Workspaces`。
2.  在 `Workspaces` 窗口空白处右键，选择 `New Workspace...`。
3.  **配置工作区**:
    *   **Workspace name**: 为工作区命名，建议格式为 `[用户名]_[项目名]_[机器名]`，例如 `John_MyUEProject_Desktop`。
    *   **Workspace root**: 选择一个**本地空目录**作为工作区根目录，例如 `C:\Perforce\MyUEProject`。
    *   **Workspace Mappings**: 设置服务器仓库与本地目录的映射关系。
        *   例如，如果你的 Depot 叫 `MyUEProject`，工作区叫 `John_MyUEProject_Desktop`，映射关系应为：
          `//MyUEProject/... //John_MyUEProject_Desktop/...`

#### 第4步：配置忽略文件 (.p4ignore)

为了防止将 UE 自动生成的缓存和临时文件提交到服务器，我们需要配置一个忽略列表。

1.  在您的**工作区根目录** (例如 `C:\Perforce\MyUEProject`) 下，创建一个名为 `.p4ignore.txt` 的文本文件。
2.  将以下内容粘贴到文件中并保存：

    ```
    # Unreal Engine 4/5 ignore list
    Binaries/
    Build/
    DerivedDataCache/
    Intermediate/
    Saved/
    Script/
    .vs/
    *.VC.db
    *.suo
    *.opensdf
    *.sdf
    *.sln
    *.xcodeproj
    *.xcworkspace
    ```
3.  **告知 Perforce 忽略文件的位置**:
    *   在 P4V 菜单栏选择 `Connection -> Set P4IGNORE...`。
    *   在弹出的对话框中输入你的忽略文件名 `.p4ignore.txt`，然后确认。

#### 第5步：上传项目到 Perforce

1.  将您的虚幻引擎项目文件夹（例如 `MyProject`）**完整地移动或复制**到您的工作区根目录（例如 `C:\Perforce\MyUEProject`）下。
2.  返回 P4V，在 `Workspace` 视图中，右键点击您的项目文件夹，选择 `Mark for Add`。
3.  此操作会将所有未被忽略的文件添加到一个待提交的变更列表 (Changelist) 中。
4.  切换到 `Pending` 视图，找到这个默认的变更列表，右键点击它，选择 `Submit...`。
5.  在弹出的窗口中填写本次提交的描述（例如 "Initial project commit"），然后点击 `Submit` 按钮。文件将开始上传到服务器。

#### 第6步：在虚幻引擎中连接

1.  打开您本地工作区中的 UE 项目（例如 `C:\Perforce\MyUEProject\MyProject\MyProject.uproject`）。
2.  在编辑器主界面的右下角，点击**版本控制图标**（未连接时通常显示 "Source Control"）。
3.  在弹出的菜单中选择 `Connect to Source Control...`。
4.  在提供商 (Provider) 下拉菜单中选择 `Perforce`。
5.  虚幻引擎通常会自动检测并填充服务器信息。请确认以下信息是否正确：
    *   **Server**: 你的服务器地址，例如 `localhost:1666`。
    *   **User Name**: 你的 Perforce 用户名。
    *   **Workspace**: 你的工作区名称，例如 `John_MyUEProject_Desktop`。
6.  点击 `Accept Settings`。

连接成功后，右下角的版本控制图标会变为绿色。此时，当您在内容浏览器中修改任何资产时，都会看到文件状态的实时更新（例如，出现红色对勾表示文件已被您签出/checkout）。

至此，配置全部完成！
