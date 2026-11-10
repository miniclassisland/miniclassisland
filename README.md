# MINI ClassIsland (Monorepo Aggregate)

<p align="center">
  <a href="https://miniclassisland.com" target="_blank" rel="noopener noreferrer">
    <img src="assets/miniclassisland.png" alt="MINI ClassIsland Logo" width="128" height="128" />
  </a>
</p>


> **MINI ClassIsland** — [ClassIsland](https://github.com/ClassIsland/ClassIsland) 企业级/校园级多租户集中管控生态系统  
> 完整兼容上游 [Cyrene_MSP 协议 v2.0.0.0](https://github.com/ClassIsland/ClassIsland/tree/master/doc/Design/Cyrene_MSP)，专为千级租户、万级并发终端场景设计。

[![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-green)](core/LICENSE)
[![Cyrene_MSP: v2.0.0.0](https://img.shields.io/badge/Cyrene__MSP-v2.0.0.0-indigo)](https://github.com/ClassIsland/ClassIsland)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](core/)
[![Next.js 16](https://img.shields.io/badge/Next.js-16.2-black)](web/)
[![Android SDK 35](https://img.shields.io/badge/Android-API%2026--35-3DDC84)](android/)
[![HarmonyOS Next](https://img.shields.io/badge/HarmonyOS%20Next-API%2012%2B-0A59F7)](hmos/)

---

## 1. 项目概览

**MINI ClassIsland** 是一套面向中小学校园、教育集团及教研室的多租户 ClassIsland 终端集中管理（集控）平台。本仓库为聚合根仓库（Aggregate Monorepo），通过 Git Submodules 统一编排服务端核心、Web 租户工作台、超级管理员后台以及原生移动端（Android / HarmonyOS Next）控制客户端，并提供生产环境 Nginx 网关配置与公共字体资源。

### 子仓库矩阵

| 模块路径 | 仓库名称 | 角色定位 | 核心技术栈 | 文档入口 |
| :--- | :--- | :--- | :--- | :--- |
| [`core/`](core/) | `miniclassisland/core` | 集控核心后端（HTTP 三端口 + gRPC 服务端 + CLI） | Python 3.10+, FastAPI, gRPC (`grpcio`), SQLAlchemy 2.0 (Async), PostgreSQL, Redis, PGPy | [README](core/README.md) · [CONTRIBUTING](core/CONTRIBUTING.md) |
| [`web/`](web/) | `miniclassisland/web` | 租户管理与教务集控工作台（`dash.miniclassisland.com`） | Next.js 16, React 19, TypeScript 5, Fluent UI React v9 | [README](web/README.md) · [CONTRIBUTING](web/CONTRIBUTING.md) |
| [`admin/`](admin/) | `miniclassisland/admin` | 超级管理员全局治理控制台（`admin.miniclassisland.com`） | Next.js 16, React 19, TypeScript 5, Fluent UI React v9 | [README](admin/README.md) · [CONTRIBUTING](admin/CONTRIBUTING.md) |
| [`android/`](android/) | `miniclassisland/android` | 原生 Android 移动/平板集控端（`org.miniopensource.miniclassisland`） | Java 17 / Android SDK 35, 自适应手机与平板双栏 UI | [README](android/README.md) · [CONTRIBUTING](android/CONTRIBUTING.md) |
| [`hmos/`](hmos/) | `miniclassisland/hmos` | 原生 HarmonyOS Next 多设备集控端（`org.miniopensource.miniclassisland`） | ArkTS, HarmonyOS SDK API 12+ (`26.0.0`), `freeMultiWindow` 自由多窗 | [README](hmos/README.md) · [CONTRIBUTING](hmos/CONTRIBUTING.md) |
| [`nginx/`](nginx/) | *(聚合仓库内置)* | 统一反向代理、泛域名多租户路由与 HTTP/2 + gRPC 网关 | Nginx (TLS 1.2/1.3, HTTP/2, `grpc_pass`) | [`nginx/miniclassisland.conf`](nginx/miniclassisland.conf) |
| [`assets/`](assets/) | *(聚合仓库内置)* | 跨端共享设计资产与开源字体（HarmonyOS Sans 等） | 静态资源 | - |

---

## 2. 系统架构与多端口/泛域名拓扑

MINI ClassIsland 采用**四端口职责隔离 + 泛域名租户路由**架构，确保终端设备高频拉取与管理面 API 互不干扰，并在网关层实现严格的权限边界：

```
                           ┌───────────────────────────────────────────────┐
                           │           Nginx TLS 1.3 / HTTP/2 网关          │
                           └───────┬───────────────┬───────────────┬───────┘
                                   │               │               │
       ┌───────────────────────────┘               │               └───────────────────────────┐
       ▼                                           ▼                                           ▼
 *.miniclassisland.com                 dash.miniclassisland.com                   admin.miniclassisland.com
 (泛域名租户终端接入)                    api.miniclassisland.com                    (超管前端 + /api 反代)
       │                                           │                                           │
       ├── HTTP GET /api/v1/client/* ──► :27041    ├── HTTP REST ─────────────► :27042         └── HTTP REST ──► :27043
       │   (Client API: Manifest & 资源)            │   (Management API: 租户集控)                  (Admin API: 全局超管)
       │                                           │
       └── application/grpc ───────────► :27044    └── Android / HarmonyOS 原生客户端直连
           (Cyrene_MSP gRPC: 注册/握手/命令流/日志/审计)
```

### 端口矩阵

| 端口 | 服务模块 | 绑定域名 / 路由 | 核心职责 |
| :---: | :--- | :--- | :--- |
| **`:27041`** | **Client API** | `https://<slug>.miniclassisland.com` | 面向教室内 ClassIsland 终端：下发配置清单（`Manifest`）与 7 类标准化资源 JSON（`/get`）。内置内存级规范化资源缓存（`_NORMALIZED_RESOURCE_CACHE`）与终端 IP 强校验。 |
| **`:27042`** | **Management API** | `https://api.miniclassisland.com` | 面向学校管理员/教务老师：OOBE 状态查询、注册登录、2FA (TOTP/恢复码)、租户创建、7 类档案可视化编辑、终端准入审批（开放/配对码/白名单）、实时指令与广播推送、日志提取与审计查询。 |
| **`:27043`** | **Admin API** | `https://admin.miniclassisland.com/api` | 面向平台超级管理员：系统首次 OOBE 初始化、全局 $O(1)$ 实时监控指标（`/stats`）、全平台用户启停、注册审批队列、租户配额管控（`AccountQuota`）、一键 SSO 免密接管租户集控（`进入集控编辑`）。 |
| **`:27044`** | **Cyrene_MSP gRPC** | `grpcs://<slug>.miniclassisland.com:443` | 面向 ClassIsland 终端的实时双向通道：`ClientRegister`（注册与 PGP 密钥下发）、`Handshake`（OpenPGP 签名验签与挑战握手）、`ClientCommandDeliver`（`ListenCommand` 双向流心跳与命令投递）、`ConfigUpload`（高达 2MB 的客户端配置与 1000 条运行日志快照上传）、`Audit`（审计事件上报）。 |

---

## 3. 核心架构亮点与高并发优化

1. **千级租户、万级连接扩展能力**：
   - **数据库层**：全部多租户配置表（`ClassPlan`、`TimeLayout`、`Subjects`、`DefaultSettings`、`Policy`、`Components`、`Credentials`、`ClientProfile`）均采用 `(account_id, name)` 复合主键与索引，彻底消除跨租户全表扫描。
   - **内存级规范化配置缓存**：`:27041/get` 接口以 `(account_id, resource_type, name, version)` 为键缓存规范化后的 JSON 字节流，2,000 台终端并发拉取 ~188 KB 全量配置时零重复 JSON 反序列化开销，P50 延迟低至 ~25ms。
   - **$O(1)$ 全局与租户在线状态统计**：超管面板 `/stats` 与 `ListenCommand` 配额检查直接读取内存活跃队列字典（`client_queues` / `tenant_clients`），响应耗时 `< 2 ms`，避免大规模租户下的 Redis `SCAN`/`HGETALL` 往返风暴；Redis 会话状态同步采用 `pipeline(transaction=False)` 批量合并。
2. **严密的安全与防篡改体系**：
   - **防伪造真实 IP 解析**：内置可信代理白名单（`_TRUSTED_PROXY_HOSTS`），拒绝空字符串 Host 及非授信来源伪造的 `X-Forwarded-For` / `X-Real-IP`。
   - **终端身份与 IP 双向绑定**：`CompleteHandshake` 校验已注册终端绑定 IP，防止攻击者劫持 `ClientUid` 覆写终端 IP 并窃取 `Manifest` 中的管理凭据哈希；管理员可通过后台显式重置迁移设备的绑定 IP。
   - **配置与日志上传鉴权加固**：`ConfigUpload.UploadConfig` 兼容上游无 Session Metadata 的实现，通过 `RequestGuidId` 状态机 + 终端会话/绑定 IP 联合校验，杜绝匿名伪造快照覆盖。
   - **原子化一次性令牌**：SSO 跨站兑换码与终端下载令牌均基于 Redis Lua 脚本（`GET` + `DEL`）原子消费，彻底防御并发重放攻击。
   - **安全响应头与 HTML 转义**：全线启用 CSP、HSTS、`X-Frame-Options: DENY`、`X-Content-Type-Options: nosniff`，邮件通知模板全量 `html.escape` 防护 HTML 注入。

---

## 4. 快速开始

### 4.1 克隆聚合仓库与子模块

```bash
git clone --recurse-submodules https://github.com/miniclassisland/miniclassisland.git
cd miniclassisland

# 若已克隆但未拉取子模块：
git submodule update --init --recursive
```

### 4.2 启动 Core 后端服务

环境要求：Python 3.10+（推荐 3.12）、[uv](https://docs.astral.sh/uv/)、PostgreSQL 14+、Redis 6+。

```bash
cd core
uv sync --all-groups

# 生成 Protobuf / gRPC 代码（如需更新）
uv run miniclassisland ci grpc generate

# 启动守护进程（同时监听 :27041, :27042, :27043, :27044）
uv run miniclassisland sys daemon
```

### 4.3 启动或构建 Web 与 Admin 前端

环境要求：Node.js 20+、npm。

```bash
# 租户集控面板 (web)
cd web
cp .env.example .env.local
npm install
npm run dev      # 开发模式
npm run build    # 导出静态站点至 web/out/

# 超级管理员后台 (admin)
cd ../admin
cp .env.example .env.local
npm install
npm run dev      # 开发模式
npm run build    # 导出静态站点至 admin/out/
```

### 4.4 构建原生移动端应用

- **Android 客户端 (`android/`)**：
  ```bash
  cd android
  ./build-apk.bat   # Windows 下自动编译并签名输出 APK 至 android/dist/
  ```
- **HarmonyOS Next 客户端 (`hmos/`)**：
  ```bash
  cd hmos
  ./build-hap.bat   # 使用 DevEco Studio 工具链编译输出 HAP 至 hmos/dist/
  ```

---

## 5. 生产环境部署（Systemd + Nginx）

1. **后端 Systemd 服务**：将 `core` 配置为 `miniclassisland-backend.service`，工作目录指向 `core/`，首次完成 OOBE 初始化后系统配置持久化于 `core/.miniclassisland/config.json`。
2. **前端静态产物**：分别在 `web/` 与 `admin/` 执行 `npm run build`，由 Nginx 直接托管 `web/out` 与 `admin/out`。
3. **Nginx 网关配置**：参考 [`nginx/miniclassisland.conf`](nginx/miniclassisland.conf)，配置泛域名 TLS 证书（覆盖 `miniclassisland.com` 与 `*.miniclassisland.com`），启用 HTTP/2 与 `@grpc` 分流转发。

---

## 6. 贡献指南与开源协议

- 在提交代码或 Pull Request 前，请仔细阅读根目录 [`CONTRIBUTING.md`](CONTRIBUTING.md) 以及对应子模块的 `CONTRIBUTING.md`。
- **强制签名提交**：本项目及所有子仓库要求所有 Git Commit 必须使用在 GitHub 登记过的 SSH 密钥或 GPG 密钥进行签名（`git commit -S`）。
- 本项目基于 [GNU General Public License v3.0 (GPL-3.0)](core/LICENSE) 开源。
