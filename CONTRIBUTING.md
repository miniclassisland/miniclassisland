# 贡献指南 (Contributing to MINI ClassIsland)

感谢你对 **MINI ClassIsland** 项目的关注与贡献！本文件规定了聚合根仓库（`miniclassisland`）及旗下五个子仓库（`core`、`web`、`admin`、`android`、`hmos`）通用的协作规范、质量门禁与提交签名要求。

---

## 1. 仓库结构与子模块工作流

本项目采用 **Git Submodules** 组织多端代码：

- **根仓库 (`miniclassisland`)**：维护子模块版本指针（`.gitmodules`）、生产环境 Nginx 配置（[`nginx/miniclassisland.conf`](nginx/miniclassisland.conf)）、公共字体资源（`assets/`）与全局忽略规则（[`.gitignore`](.gitignore)）。
- **子仓库**：
  - [`core/`](core/CONTRIBUTING.md) — Python 集控核心后端
  - [`web/`](web/CONTRIBUTING.md) — Next.js 租户管理前端
  - [`admin/`](admin/CONTRIBUTING.md) — Next.js 超级管理员前端
  - [`android/`](android/CONTRIBUTING.md) — 原生 Android 客户端
  - [`hmos/`](hmos/CONTRIBUTING.md) — 原生 HarmonyOS Next 客户端

### 跨子模块修改提交流程

1. **先在子模块内完成开发、验证与提交**：
   进入对应的子目录（例如 `cd core`），切换到目标分支（通常为 `main` 或功能分支），通过全部测试与代码检查后执行签名提交。
2. **再回到根仓库更新子模块指针**：
   当子模块提交完成后，回到根目录 `miniclassisland` 提交更新后的子模块 commit 引用。

---

## 2. 强制要求：SSH / GPG 提交签名 (`git commit -S`)

为了保障供应链安全与提交者身份可信，**本项目及所有子仓库要求每一笔 Commit 必须附带经过 GitHub 验证的加密签名（优先使用 SSH 密钥签名）**。

### 配置本地 SSH 签名

```bash
# 全局或在仓库内启用 SSH 签名格式
git config --global gpg.format ssh
git config --global user.signingkey "~/.ssh/id_ed25519.pub"
git config --global commit.gpgsign true
```

提交时请使用 `-S` 参数（若已设置 `commit.gpgsign=true` 则自动启用）：

```bash
git commit -S -m "feat(core): optimize global stats and manifest caching"
```

提交后可通过以下命令验证签名状态：

```bash
git log --show-signature -n 1
```

---

## 3. Commit Message 规范

本项目遵循 **Conventional Commits（约定式提交）** 规范：

```text
<type>(<scope>): <subject>
```

- **常用 `type`**：
  - `feat`：新增功能或协议支持
  - `fix`：修复 Bug、竞态条件或安全缺陷
  - `perf`：性能优化（如缓存、SQL/Redis 批量化、并发吞吐提升）
  - `sec`：安全加固与鉴权修复
  - `docs`：文档（`README.md`、`CONTRIBUTING.md`、注释）更新
  - `test`：新增或重构自动化测试与压测脚本
  - `chore` / `build`：构建脚本、依赖项、`.gitignore` 或 Nginx 配置调整
- **常用 `scope`**：`core`、`grpc`、`client`、`management`、`admin`、`web`、`android`、`hmos`、`nginx`、`docs`。

---

## 4. 安全与敏感信息防护准则

1. **严禁提交任何密钥、证书或环境文件**：
   - 严禁将 `.env`、`.env.local`、`.miniclassisland/config.json`、私钥（`*.pem`、`*.key`、`*.pfx`、`*.p8`、`id_rsa*`、`id_ed25519*`、`id_ecdsa*`）、证书（`*.crt`）或本地数据库快照（`*.db`、`*.sqlite*`、`dump.rdb`）纳入版本控制。
   - 新增配置项时，请同步更新对应的 `.env.example` 模板。
2. **多租户隔离与鉴权检查**：
   - 任何涉及租户资源读写的代码必须显式携带 `account_id` 过滤条件，严禁仅凭 `name` 或 `client_id` 单字段查询。
   - 终端侧接口（`:27041` 与 `:27044`）必须严格校验终端绑定 IP 或有效会话（`session` + `cuid`）。

---

## 5. 各子仓库质量门禁汇总

在发起 Pull Request 前，必须确保所修改模块的检查全部通过：

| 模块 | 必跑验证命令 | 准入标准 |
| :--- | :--- | :--- |
| **`core/`** | `uv run pytest --cov=app --cov-report=term-missing`<br>`uv run black --check app tests`<br>`uv run ruff check app tests`<br>`uv run pyright` | **测试覆盖率 100% (`0 Miss`)**，格式、Lint、类型检查零错误 |
| **`web/`** | `npx tsc --noEmit`<br>`npm run lint`<br>`npm run build` | TypeScript 零报错，ESLint 通过，静态导出构建成功 |
| **`admin/`** | `npx tsc --noEmit`<br>`npm run lint`<br>`npm run build` | TypeScript 零报错，ESLint 通过，静态导出构建成功 |
| **`android/`** | `build-apk.bat` 或 `./gradlew assembleDebug` | 编译打包无报错，手机与平板双栏自适应布局正常 |
| **`hmos/`** | `build-hap.bat` 或 `hvigorw assembleHap` | ArkTS 编译通过，多窗口断点（`sm`/`md`/`lg`）渲染正常 |
