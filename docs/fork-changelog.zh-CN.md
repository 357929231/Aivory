# 本 Fork 新增功能清单

> **基线**：`hjxwz123/Aivory` 的 `f945624`（`chore: prepare v2.4.9 beta.6 prerelease`）
> **本 Fork 相对基线新增两批功能**，共 2 个提交、149 个文件。
> 本文档面向"这份代码比原版多了什么、怎么用、边界在哪"，实现细节见各功能自己的设计文档。

---

## 一、总览

| # | 功能 | 提交 | 主要文件 |
| --- | --- | --- | --- |
| 1 | 右侧预览面板统一 + 可拖动分隔条 + 文档编辑器 | `35b5cd1` | `src/components/chat/artifact-panel.tsx`、`src/lib/artifact-panel-width.ts`、`src/components/files/editors/*` |
| 2 | 上传整个目录，让 AI 在目录里执行任务 | `35b5cd1` | `src/lib/folder-upload.ts`、`src/lib/folder-attachments.ts`、`server/internal/api/upload_policy.go`、`server/internal/tools/builtins.go` |
| 3 | AI PPT（文多多 Docmee）+ 积分计费 | `7ea4722` | `server/internal/api/docmee_handlers.go`、`src/pages/ppt/AiPPT.tsx`、`src/lib/aippt-billing.ts` |

---

## 二、功能 1：右侧预览面板统一 + 可拖动分隔条 + 文档编辑器

### 2.1 新增能力

| 能力 | 说明 |
| --- | --- |
| **统一右侧面板** | 原本"HTML 产物预览"和"文件预览"是两个各自独立的界面，现在合并为一个 `ArtifactPanel`，流式 HTML 与打开的文档共用同一个槽位 |
| **可拖动分隔条（新）** | 面板左边缘可拖动 / 可用键盘调整宽度：向左拉宽、向右收窄，**HTML 预览、Word/Excel/PPT 文档、编辑器一起跟着变** |
| **宽度持久化（新）** | 宽度写入 `aivory.settings`，刷新/重开保持；**为对话区保留 360px**，窗宽变窄时自动收紧，绝不把对话挤出屏幕 |
| **键盘操作（新）** | 聚焦分隔条后 `←/→` 每次 16px、`Home`/`End` 到两端、`Enter` 或双击恢复默认；带 `role="separator"` + `aria-valuenow` |
| **四类文档编辑器** | 代码（CodeMirror）、Word、Excel、PowerPoint 可编辑，各自按需懒加载（`@docx-editor.dev`、AG Grid、自研 OOXML 外科手术写回） |
| **编辑器错误边界（新）** | 预览与编辑区都包在 `EditorErrorBoundary` 里：编辑器抛错时降级为「此文档无法打开编辑」+ 重试按钮，而不是整块空白 |
| **编辑区按文件重建（新）** | 编辑区以 `<文件名>:edit` 为 key，切换文档时彻底重建，不再把新文档交给仍持旧状态的编辑器实例 |

### 2.2 修掉的真实缺陷

**"点击「编辑」显示空白"** —— 根因是应用里**没有任何 error boundary**：编辑区一旦抛错，整棵 React 树被卸载，用户看到的就是纯空白（无工具栏、无报错、无任何可操作项）。现已在预览与编辑两处都加了边界。

同时加固了 `--chat-side-panel-width` 变量与开合动画的配合：面板展开/收起动画期间动画值仍然权威，拖动不会和它打架。

### 2.3 使用方式

打开任意 HTML 产物或文档 → 右侧面板出现 → **鼠标拖动面板左边缘**（或聚焦后按方向键）即可改变宽度 → 点面板标题栏的铅笔图标进入编辑、眼睛图标回到预览。

---

## 三、功能 2：上传整个目录，让 AI 在目录里执行任务

### 3.1 新增能力

| 能力 | 说明 |
| --- | --- |
| **文件夹上传（新）** | 输入框旁「上传文件夹」（桌面在工具栏，移动端在「+」菜单）。选中本地文件夹后整个目录上传 |
| **目录结构保留（新）** | 每个文件带 `folder_name` + `rel_path`，**结构一路保留到代码沙箱**：`/workspace/uploads/<目录>/<相对路径>`，不再是以前那种被拍平的布局 |
| **浏览器侧预过滤（新）** | 自动跳过依赖/构建目录（`node_modules`、`.git`、`dist`、`__pycache__` …）、系统噪声文件、管理员扩展名白名单之外的文件，并受数量（300）与体积上限约束 |
| **如实告知跳过了什么（新）** | 跳过项按原因分类计数并弹出提示，**不静默丢弃** |
| **目录清单喂给模型（新）** | 额外在沙箱写一份 `/workspace/aivory-uploads.txt`（含每个文件大小、排序、超 2000 条时提示用 `os.walk`），并改写 `python_execute` 说明，让模型处理"项目/目录"类问题先读清单 |
| **附件区目录折叠（新）** | 整个目录显示为**一个节点**（目录名 + 文件数 + 体积 + 聚合上传进度），可展开看成员相对路径，可单独移除成员或整目录；成员仍是普通附件，**发送不受影响**，刷新后靠 `rel_path` 保持折叠 |

### 3.2 AI 侧怎么用

上传目录后直接提问，模型会：

1. 读 `/workspace/aivory-uploads.txt` 拿到目录全貌（或 `for r, d, fs in os.walk('/workspace/uploads')` 自行枚举）；
2. 在沙箱里对**真实路径**操作（`open()` / `grep` / pandas / python-docx …）；
3. 产物写到 `/workspace/outputs`，自动作为可下载附件回到对话。

**降级行为**：未配置沙箱（无 Docker / `SANDBOX_BASE_URL` 为空）时 `python_execute` 不可用，但目录文件仍完整入库、可预览、可下载，**不报错、不丢数据**。

### 3.3 设计取舍（为什么这么做）

- **目录内容不入 RAG**：一个项目几百个源文件，逐一「解析→分块→嵌入」会把入库队列和向量库打满，而模型根本不需要检索——它直接在沙箱里读。单文件上传的两条入库路径**逐字未动**。
- **逐文件请求，不打包 zip**：复用现成上传链路（配额 / 权限 / 白名单 / 限流 / 路径越界防护全都在），失败粒度也更细；用前端过滤 + 并发 3 兜住请求数。
- **清单放在 `uploads/` 之外**：沙箱每次调用会清空 `/workspace/uploads`，放里面会被清掉；放外面则每轮重写，且模型 glob 时不会把清单当成用户的文件。

### 3.4 安全边界

`rel_path` 是**不可信输入**且最终会变成沙箱路径，因此：

- 接口入口拒绝绝对路径、`..`、空段、`.`、控制字符、超长/超深路径；
- 校验「`rel_path` 末段 == 已校验文件名」，防止客户端描述与实体不符；
- 沙箱侧 `PutFile` 由 sidecar 再做 `_safe_under_workspace` + `realpath` 复核；
- staging 兜底：异常 `rel_path` 直接退化为 basename，绝不产生越界路径；
- **8 种恶意/畸形路径的测试全部返回 400**。

---

## 四、功能 3：AI PPT（文多多 Docmee）+ 积分计费

### 4.1 新增能力

| 能力 | 说明 |
| --- | --- |
| **AI PPT 入口（新）** | 侧边栏新增入口 + `/ppt` 路由（已纳入 chat shell，导航缓存与实时流行为与其他分区一致） |
| **内嵌创作台（新）** | 内嵌 Docmee 演示文稿创作台 iframe |
| **密钥不出服务端（新）** | 服务端代理 token 接口，**浏览器始终拿不到 Docmee API Key**：`/me/ppt/config`、`/me/ppt/token`、`/me/ppt/attempt`、`/me/ppt/charge`、`/me/ppt/release` |
| **积分计费（新）** | **先预扣、后结算**：`attempt` 先占用积分，余额不足即失败关闭；`charge` 保证一份 PPT 只计费一次；`release` 归还未出成果的预扣；`charge`/`release` 拒绝操作他人的 attempt |
| **配置门控（新）** | 未配置时接口返回明确的「未配置」错误，而不是暴露半可用界面；`config` 只回传价格、**不含任何密钥**；上游失败不泄露 provider 细节 |
| **管理端设置（新）** | API Key、API base URL、domain、SDK URL、creator 版本、token 有效期、每份扣费、启用开关；复用既有 setting 存储往返并对 URL 归一化；API Key 以掩码回显，回传掩码即"保留已存值" |
| **用量与计费可见（新）** | 每次成功计费写入一条 `usage_logs` 记录（purpose = `ppt`），供管理后台「用量与计费」列表与汇总统计读取；以 PPT id 为幂等键，SDK 重试 / 刷新重报 / 重开 attempt **都不会重复计账**；若首次写入失败，后续重放会**自动补写**而不会让该 PPT 永久消失 |
| **五语言文案（新）** | 新增 `ppt` namespace，中/繁/英/日/法齐全；管理端用量页新增 `ppt` 用途标签与筛选项 |

### 4.2 使用方式

管理端「积分设置」→ 配置 Docmee API Key 与每份扣费 → 打开启用开关 → 侧边栏进入 **AI PPT** → 在创作台里生成演示文稿，每生成一份按配置扣积分。

---

## 五、验证状态

| 范围 | 结果 |
| --- | --- |
| 前端全量 `vitest run --dir tests/frontend` | **136 文件 / 804 用例全通过** |
| Go `internal/api`（Docmee 专项） | **10 个用例全过**（token 铸造与缓存、预扣/结算/归还语义、无额度失败关闭、积分关闭时免费、拒绝他人作业、配置门控、不泄露密钥、管理端往返与 URL 归一化） |
| Go `internal/store` | 通过（564s） |
| Go `internal/tools` | 通过（142s） |
| Go `internal/api` 全量 | 仅 2 项失败，均为 **Windows 平台固有**且发生在本次未修改的函数上：`os.Stat().Mode().Perm()` 在 Windows 恒为 `0666`；末尾检查 `/proc/self/environ` |
| `tsc -b --noEmit` | 0 error |
| `go vet ./...` | 干净 |
| `vite build` | 通过 |
| 浏览器回归 | `npm run test:panel`（真实 Chrome：文档预览、预览⇄编辑往返、分隔条拖动/键盘/持久化/刷新恢复、markup 编辑与实时预览）、`npm run test:preview`、`npm run test:browser` 全过 |

### 本机环境备注

- **默认 `GOARCH=386` 且 32 位 mingw 链接器损坏**，`go build`/`go test` 的 386 目标无法链接。用仓库自带的 64 位工具链：
  `.tools/go1.26.6/golang.org/toolchain@v0.0.1-go1.26.6.windows-amd64/bin/go.exe`，并设 `GOARCH=amd64`、`CGO_ENABLED=1`（sqlite 驱动需要 cgo）。
- **`go test ./...` 一把跑会超时**（store 包在慢盘上约 565s，默认超时 600s）：请分包跑或加 `-timeout`。这是环境性能问题，不是用例失败。

---

## 六、明确未覆盖 / 已知边界

| 项 | 现状 | 影响 |
| --- | --- | --- |
| 右侧「会话文件」抽屉仍是平铺列表 | 未按 `rel_path` 建树（附件区已折叠） | 抽屉里 300 个文件仍是 300 行；`rel_path` 已在接口里，抽屉建树是纯渲染层后续工作 |
| 空目录不上传 | 浏览器目录选择器**只返回文件**、不返回空目录 | 空目录结构会丢失，Web API 既有限制 |
| 符号链接 | 按文件返回，指向的目标被当普通文件上传 | 无安全影响，可能重复上传同一内容 |
| 沙箱不可用时不能执行 | 需在部署侧配置沙箱服务 | 代码路径已按"有则用、无则降级"实现 |
| 目录上传是逐文件请求 | 300 个文件 = 300 次请求 | 受控并发压到 3，弱网下仍慢；将来可考虑批量端点 |
| 管理端用量表格不显示「积分」列 | `AdminUsageRecord` 未取 `usage_logs.credits` | 列表看得到用途/模型/成本，但逐行积分需看计费汇总或用户积分明细；补列是纯展示工作 |

---

## 七、已修复的真实缺陷（交付后反馈）

| 缺陷 | 根因 | 修法 |
| --- | --- | --- |
| **AI PPT 生成后没有扣费记录，用量与计费里不显示** | 结算积分预扣只写 `credit_ledger`，而管理端「用量与计费」**只读 `usage_logs`**（汇总读 `usage_stats`，由数据库触发器从 `usage_logs` 镜像）。Docmee 从不写这两张表，所以积分确实扣了、账本有记录，用量报表却一行都没有 | 每次成功计费写一条 `usage_logs`（purpose = `ppt`，credits = 实际结算额）；以 PPT id 作幂等键，重放/刷新/重开 attempt 不重复计账；首次写入失败时后续重放自动补写；管理端新增 `ppt` 用途文案与筛选项 |

> 该缺陷的回归测试见 `server/internal/api/docmee_usage_test.go`（5 个用例：行写入与汇总口径、重放不重复、重开 attempt 不重复、两份额度两条记录、丢失写入可补写、未计费不产生记录）。

---

## 八、相关设计文档

| 文档 | 内容 |
| --- | --- |
| `docs/folder-upload.zh-CN.md` | 目录上传的决策、改动位置、边界与取舍、验证记录 |
| `docs/document-preview-and-edit.zh-CN.md` | 文档预览与编辑的技术选型、分阶段实现、真实缺陷排查记录 |
| `docs/ai-ppt-docmee.md` | AI PPT / Docmee 集成方案 |
