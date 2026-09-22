# Aivory 目录上传与「在目录里执行任务」— 实现记录

> 需求原文：**「希望该系统支持上传整个目录功能，实现上传目录去让 AI 在目录里面执行任务」**
> 本文记录已确认的决策、改动点（含文件与行号级别的位置）、验证方式，以及**未覆盖的边界**。
> 版本基线：v2.4.9-beta.6。

---

## 〇、已拍板的决策

| # | 决策 | 结论 |
| --- | --- | --- |
| 1 | 能力边界 | **目录上传 + 目录树清单 + 沙箱执行**。有沙箱时 `python_execute` 真跑代码；无沙箱时降级为目录树 + 检索 + 预览，功能不报错 |
| 2 | 上传策略 | **前端白名单过滤 + 每文件上限 + 总量上限**，并且**明确告知用户跳过了什么**，不静默丢弃 |
| 3 | 请求形态 | **一个文件一个请求**（复用 `POST /api/files`），不做 zip 打包、不加新表 |
| 4 | 结构存储 | files 表新增 `rel_path` 列，记录「文件在目录内的相对路径」 |
| 5 | RAG | **目录内容不做 RAG 入库**（见 §3.1 理由） |
| 6 | 沙箱落地 | `/workspace/uploads/<目录名>/<相对路径>`，并写一份清单到 `/workspace/aivory-uploads.txt` |

---

## 一、为什么这样设计

### 1.1 一个文件一个请求

目录上传有两种做法：打包成 zip 传一次，或逐文件上传。

选**逐文件**，理由：

- 现有链路已经完整且经过安全加固（配额、权限、白名单、限流、`fileguard` 路径越界防护）。zip 方案等于在服务端新开一条解压链路，需要重新处理 zip-slip、解压炸弹、配额记账，风险与工作量都不划算。
- 失败粒度更好：一个文件超限不会毁掉整个目录。
- 已有能力直接复用：附件展示、预览、删除、分支归属（`branch_message_id`）全都自动生效。

代价是请求数多，用三处机制兜住：

1. **前端预过滤**（`src/lib/folder-upload.ts`）——`node_modules`/`.git`/`dist` 等目录、系统噪声文件、白名单外扩展名在浏览器侧就丢掉；
2. **受控并发**——一次 3 个请求（`handleAttach(files, 3)`），既压满慢链路又不会打爆服务端每用户上传限流；
3. **服务端限流**——沿用既有的每小时/每分钟预算，超限返回 429，前端按文件报错而不是整体失败。

### 1.2 结构为什么必须落库

`rel_path` 不是"上传时的装饰"，而是**后续每一轮对话都要读的持久状态**：沙箱 staging 与目录清单都在**下一次** `python_execute` 调用时才生成，那时原始请求早已结束。所以结构必须存进数据库，不能只留在前端内存里。

存储选择：files 表加一列，而不是新建 `file_folders` 表。

- 复用了已有的分支可见性（`ListFilesByConversationBranch`）、配额记账、删除级联；
- 新表意味着重写这些语义，收益为零。

列定义（`rel_path TEXT NOT NULL DEFAULT ''`）在 SQLite 与 Postgres 两侧的建表语句与增量迁移里都补齐了，老库靠增量迁移补列，`''` 就是"单文件上传"的历史语义。

### 1.3 沙箱清单为什么是"文件"而不是"工具"

模型要"在目录里执行任务"，第一步是**知道目录里有什么**。三条可选路径：

- 只在 tool description 里说"用 `os.walk` 自己走"——模型容易漏、每次都多花一轮；
- 新增一个 `list_folder` 工具——多一个工具就要多一份权限/白名单/预算维护；
- **上传时就写一份清单文件**（本方案）——零新增工具、零新增权限面，模型一次 `read` 或一次 `print` 就能拿到全貌。

清单写在 `/workspace/aivory-uploads.txt`，**故意放在 `/workspace/uploads/` 之外**：沙箱的 `reset-inputs` 每次调用都会 `rm -rf` 掉 uploads 与 skills 再重建，放在里面会被清掉；放在外面则每次调用重写，且模型 glob 上传目录时不会把清单当成用户的文件。

---

## 二、改动清单

### 2.1 存储层（`server/internal/store/`）

| 文件 | 改动 |
| --- | --- |
| `models.go` | `File.RelPath` 字段（`json:"rel_path,omitempty"`） |
| `schema.sql` / `schema_pg.sql` | `files` 建表语句加 `rel_path` |
| `store.go` | 增量迁移 `addFileRelPath`（SQLite 版 + Postgres `IF NOT EXISTS` 版），并加入迁移列表 |
| `misc.go` | `CreateFile` 的 INSERT 与回读 SELECT、`ListFilesByConversation`、`ConversationFilesByIDs`、`GetFile` 四处补上 `rel_path` |

### 2.2 上传接口（`server/internal/api/`）

| 位置 | 改动 |
| --- | --- |
| `upload_policy.go` | 新增 `validateUploadRelPath`：折叠 `\`、去掉一个前导 `./`、**拒绝**绝对路径 / 空段 / `.` / `..` / 控制字符 / 超长段 / 超深路径 |
| `files_handlers.go` | `uploadFileHandler` 读取 `folder_name` + `rel_path`，校验两者一致（`rel_path` 最后一段必须等于已校验的文件名），写入 `File.RelPath` |
| `files_handlers.go` | 新增 `folderRelativeName`：把目录名拼进持久路径，且不会出现 `folder/folder/...` 双前缀 |
| `files_handlers.go` | 目录上传**跳过**两条 RAG 入库路径（§4.14 项目 KB 自动入库、§4.11.2 会话临时文档） |
| `conversations_handlers.go` | `convFile` DTO 暴露 `rel_path`，供前端按目录聚合 |

请求字段（都是 multipart 普通字段，和文件一起提交）：

| 字段 | 含义 | 说明 |
| --- | --- | --- |
| `file` | 文件本体 | 同现有上传 |
| `folder_name` | 目录名（单个路径段） | 例如 `my-project`；不是路径，会被 `filepath.Base` 归一 |
| `rel_path` | 文件在目录内的相对路径 | 例如 `my-project/src/main.ts`；**必须**与 `folder_name` 同时出现 |

### 2.3 AI 可用性（`server/internal/tools/builtins.go`）

| 位置 | 改动 |
| --- | --- |
| `sandboxUploadPath` | 有 `rel_path` 就按原路径 staging；无则保持历史的 `uploads/<filename>` 扁平布局。同名不同目录**互不覆盖**，真正同路径冲突才改名 `-2` |
| `buildFolderManifest` | 生成目录清单（排序、含大小、超过 2000 条时截断并提示用 `os.walk` 枚举）；没有任何目录文件时返回空串 |
| `folderManifestPath` | `const folderManifestPath = "/workspace/aivory-uploads.txt"` |
| `artifactDedupeName` | 生成图与上传文件共用 `seen` 集合，避免生成图覆盖用户同名文件 |
| tool description | 明确告知模型：目录上传保留结构、清单在 `/workspace/aivory-uploads.txt`、处理"项目/目录"类问题先读清单 |

**降级行为**：沙箱未配置时 `python_execute` 不可用（沿用既有 `Enabled()` 判定），此时目录文件仍然完整入库、可预览、可检索（若管理员放开单文件 RAG），只是不能真跑代码。**不会报错、不会丢数据。**

### 2.4 前端（`src/`）

| 文件 | 改动 |
| --- | --- |
| `lib/folder-upload.ts` | **新增**。纯函数 `selectFolderFiles`：跳过目录 / 噪声文件 / 白名单外扩展名 / 无扩展名 / 超数量 / 超体积，并返回**每种原因的代表路径与数量**供 UI 如实展示；纯函数 `folderUploadFields`：把选择器路径转成服务端要求的 `folder_name` + `rel_path` 两个字段，与服务端校验规则一一对应 |
| `lib/folder-attachments.ts` | **新增**。纯函数 `groupAttachmentsByFolder`（按 `relPath` 首段分组）与 `chipRailItems`（把分组摊回一条保持附件顺序的渲染列表）、`pathWithinFolder`、`folderGroupSize` |
| `components/chat/folder-chip.tsx` | **新增**。折叠后的目录节点：目录名 + 文件数 + 总体积、聚合上传进度、失败计数、展开/收起、整目录移除 |
| `components/chat/composer-attachment-chip.tsx` | **新增**。从 composer 抽出的单文件/图片 chip（渲染逻辑逐字搬迁，未改行为） |
| `components/chat/composer.tsx` | 新增隐藏的 `webkitdirectory` 输入框 + 桌面工具栏按钮 + 移动端「+」菜单项（带副标题）；`handleAttach(files, inflight)` 支持受控并发；`handleFolderAttach` 组合过滤与上传；`folderPathOverrides` 在重新包装 FileList 时保住 `webkitRelativePath`；目录上传**不带 `rag=1`**；chip 轨道改为渲染 `chipRail`（目录节点 + 零散文件） |
| `api/types.ts` | `ApiConversationFile.rel_path?`，并在 `restoreConversationFile` 里还原到附件上（刷新后目录仍然折叠） |
| `i18n/locales/{en,zh,zh-Hant,ja,fr}/chat.json` | 新增 19 个文案键（上传/过滤 10 个 + 目录节点 9 个），五语言齐全 |

---

## 三、边界与取舍（如实记录）

### 3.1 为什么目录内容不入 RAG

目录往往是**一个项目**：几百个源文件、配置、脚本。逐文件走「解析 → 分块 → 嵌入」会有三个问题：

1. 把入库队列和向量库瞬间打满，而这批内容**模型根本不需要检索**——它会在沙箱里直接 `open()`/`grep`；
2. 会话临时文档（§4.11.2）的设计前提是"用户刚分享的少量文件"，几百个文件不符合该前提；
3. 用户问的是"帮我在这个项目里做某件事"，属于**代码操作**而非**知识问答**。

因此目录文件只做「入库 + 沙箱子目录 + 清单」，单文件上传的两条 RAG 路径保持原样（`rel_path == ""` 即历史行为，逐字未变）。

### 3.2 已知未覆盖

| 项 | 现状 | 影响 |
| --- | --- | --- |
| 附件区已折叠成树；**文件抽屉仍是平铺列表** | 上传完成后 composer 里整个目录显示为**一个**节点（目录名 + 文件数 + 体积 + 聚合进度），可展开看到成员的相对路径，可单独移除成员或整目录；但右侧「会话文件」抽屉（`conversation-files-panel`）仍逐个列文件，未按 `rel_path` 建树 | 抽屉里 300 个文件仍是 300 行；`rel_path` 已在接口里，抽屉建树是纯渲染层后续工作 |
| 空目录不上传 | 浏览器目录选择器**只返回文件**，不返回空目录 | 空目录结构会丢失，这是 Web API 的既有限制 |
| 符号链接 | 选择器按文件返回，符号链接指向的目标会被当作普通文件上传 | 无安全影响；只是可能重复上传同一内容 |
| 沙箱不可用时不能执行 | 本机无 Docker / 未配 `SANDBOX_BASE_URL` | 需要在部署侧配置沙箱服务；代码路径已按"有则用、无则降级"实现 |
| 上传是逐文件请求 | 300 个文件 = 300 次请求 | 受控并发压到 3，但弱网下仍慢；将来可考虑批量端点 |

### 3.3 安全边界

`rel_path` 是**不可信输入**，且最终会变成沙箱路径，因此：

- 在接口入口就严格校验（拒绝绝对路径、`..`、空段、`.`、控制字符、超长/超深）；
- 校验「`rel_path` 最后一段 == 已校验文件名」，防止客户端描述与实体不符；
- 沙箱侧 `PutFile` 由 sidecar 再做一次 `_safe_under_workspace` + `realpath` 复核（既有防护，未改动）；
- `sandboxUploadPath` 再兜底一次：异常 `rel_path` 直接退化为 basename，绝不产生越界路径。

---

## 四、验证

### 4.1 Go 单测

```sh
# 需要 64 位工具链：本机 386 版 mingw 链接器损坏（见 §5）
$env:GOARCH='amd64'; $env:CGO_ENABLED='1'
go test ./internal/api/ -run 'FolderUpload|ValidateUploadRelPath|FolderRelativeName' -timeout 900s
go test ./internal/tools/ -run 'SandboxUploadPath|BuildFolderManifest' -timeout 900s
```

覆盖：`rel_path` 端到端持久化（写入 → 回读 → 会话列表）、单文件上传保持空 `rel_path`、8 种恶意/畸形路径全部 400、目录上传不产生 RAG 文档、同名不同目录不互相覆盖、同路径冲突改名、清单排序与截断。

### 4.2 前端单测

```sh
npx vitest run --dir tests/frontend lib/folder-upload lib/locale-parity
```

覆盖：跳过规则优先级（`node_modules` 里的 `.js` 报"跳过目录"而不是"扩展名不符"）、扩展名白名单、无扩展名、数量上限、体积上限不切分文件、Windows 反斜杠路径、`folderUploadFields` 与服务端字段契约（单段路径不算目录、文件名不一致则不声称目录、绝对路径不当作目录）、五语言文案键对齐。

### 4.3 整体回归（本次实际执行）

| 范围 | 命令 | 结果 |
| --- | --- | --- |
| 前端全量 | `vitest run --dir tests/frontend` | **132 文件 / 764 用例全通过** |
| Go store | `go test ./internal/store/ -timeout 1200s` | 通过（564s） |
| Go tools | `go test ./internal/tools/` | 通过（142s） |
| Go api | `go test ./internal/api/ -timeout 1800s` | 1533s，**仅 2 项失败，均为 Windows 平台固有失败**（下详） |
| 生产构建 | `vite build` | 通过 |

**api 包的 2 项失败与本改动无关，均为"POSIX 语义在 Windows 上不成立"**：

| 失败的用例 | 原因 |
| --- | --- |
| `TestWriteUploadCopyPersistsCompleteFile` | 断言文件权限无 group/other 位（`0o077`），Windows 的 `os.Stat().Mode().Perm()` 恒为 `0666`；`writeUploadCopy` 本身未被本次改动触及 |
| `TestCleanupStoragePathCannotEscapeConfiguredRoots` | 用例末尾检查 `/proc/self/environ`，该路径在 Windows 上不存在，因此不返回 `ErrOutsideRoot` |

两者都发生在**本次未修改的函数**上，属于本机（Windows）跑 Linux 目标测试用例的既有问题。

> **另外注意**：`go test ./...` 一次性跑全部包时，store 包会超过默认 600s 超时（本机为 32 位宿主、磁盘较慢）。
> 这是**环境性能**问题而非用例失败——单独跑 store 包 564s 通过，失败时打印的是 `panic: test timed out` 而非断言失败。
> 建议：分开跑包，或加 `-timeout`。

---

## 五、本机环境备注（影响复现）

- **默认 `GOARCH=386`，且 32 位 mingw 链接器损坏**（`ld.exe: linker script file ... appears multiple times`），`go build` / `go test` 的 386 目标无法链接。
  解决办法：使用仓库内自带的 64 位工具链
  `.tools/go1.26.6/golang.org/toolchain@v0.0.1-go1.26.6.windows-amd64/bin/go.exe`，
  并设 `GOARCH=amd64`、`CGO_ENABLED=1`（sqlite 驱动需要 cgo）。
- **本机未运行沙箱服务**（无 Docker、`SANDBOX_BASE_URL` 为空），因此 `python_execute` 的**真实执行**路径无法在本机端到端验证；staging 路径与清单生成由纯函数单测覆盖。
